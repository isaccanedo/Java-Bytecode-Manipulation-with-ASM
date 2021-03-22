## A Guide to Java Bytecode Manipulation with ASM

# 1. Introdução
Neste artigo, veremos como usar a biblioteca ASM para manipular uma classe Java existente adicionando campos, adicionando métodos e alterando o comportamento dos métodos existentes.

# 2. Dependências
Precisamos adicionar as dependências ASM ao nosso pom.xml:

```
<dependency>
    <groupId>org.ow2.asm</groupId>
    <artifactId>asm</artifactId>
    <version>6.0</version>
</dependency>
<dependency>
    <groupId>org.ow2.asm</groupId>
    <artifactId>asm-util</artifactId>
    <version>6.0</version>
</dependency>
```

Podemos obter as versões mais recentes do asm e asm-util no Maven Central.

# 3. Noções básicas da API ASM
A API ASM fornece dois estilos de interação com classes Java para transformação e geração: baseado em evento e baseado em árvore.

### 3.1. API baseada em eventos
Esta API é fortemente baseada no padrão Visitor e é semelhante ao modelo de análise SAX de processamento de documentos XML. É composto, em sua essência, pelos seguintes componentes:

- ClassReader - ajuda a ler arquivos de classe e é o início da transformação de uma classe;
- ClassVisitor - fornece os métodos usados ​​para transformar a classe após a leitura dos arquivos brutos da classe;
- ClassWriter - é usado para gerar o produto final da transformação da classe.
É no ClassVisitor que temos todos os métodos de visitante que usaremos para tocar os diferentes componentes (campos, métodos, etc.) de uma determinada classe Java. Fazemos isso fornecendo uma subclasse de ClassVisitor para implementar quaisquer alterações em uma determinada classe.

Devido à necessidade de preservar a integridade da classe de saída em relação às convenções Java e o bytecode resultante, essa classe requer uma ordem estrita na qual seus métodos devem ser chamados para gerar a saída correta.

Os métodos ClassVisitor na API baseada em eventos são chamados na seguinte ordem:

```
visit
visitSource?
visitOuterClass?
( visitAnnotation | visitAttribute )*
( visitInnerClass | visitField | visitMethod )*
visitEnd
```

3.2. API baseada em árvore
Esta API é uma API mais orientada a objetos e é análoga ao modelo JAXB de processamento de documentos XML.

Ainda é baseado na API baseada em eventos, mas apresenta a classe raiz ClassNode. Esta classe serve como ponto de entrada na estrutura da classe.

# 4. Trabalhando com a API ASM baseada em eventos
Modificaremos a classe java.lang.Integer com ASM. E precisamos compreender um conceito fundamental neste ponto: a classe ClassVisitor contém todos os métodos de visitante necessários para criar ou modificar todas as partes de uma classe.

Precisamos apenas substituir o método de visitante necessário para implementar nossas alterações. Vamos começar configurando os componentes de pré-requisito:

```
public class CustomClassWriter {

    static String className = "java.lang.Integer"; 
    static String cloneableInterface = "java/lang/Cloneable";
    ClassReader reader;
    ClassWriter writer;

    public CustomClassWriter() {
        reader = new ClassReader(className);
        writer = new ClassWriter(reader, 0);
    }
}
```

Usamos isso como base para adicionar a interface Cloneable à classe Inteiro de estoque e também adicionamos um campo e um método.

### 4.1. Trabalhando com Campos
Vamos criar nosso ClassVisitor que usaremos para adicionar um campo à classe Integer:

```
public class AddFieldAdapter extends ClassVisitor {
    private String fieldName;
    private String fieldDefault;
    private int access = org.objectweb.asm.Opcodes.ACC_PUBLIC;
    private boolean isFieldPresent;

    public AddFieldAdapter(
      String fieldName, int fieldAccess, ClassVisitor cv) {
        super(ASM4, cv);
        this.cv = cv;
        this.fieldName = fieldName;
        this.access = fieldAccess;
    }
}
```

A seguir, vamos substituir o método visitField, onde primeiro verificamos se o campo que planejamos adicionar já existe e definimos um sinalizador para indicar o status.

Ainda temos que encaminhar a chamada do método para a classe pai - isso precisa acontecer porque o método visitField é chamado para todos os campos da classe. Deixar de encaminhar a chamada significa que nenhum campo será escrito para a classe.

Este método também nos permite modificar a visibilidade ou o tipo dos campos existentes:

```
@Override
public FieldVisitor visitField(
  int access, String name, String desc, String signature, Object value) {
    if (name.equals(fieldName)) {
        isFieldPresent = true;
    }
    return cv.visitField(access, name, desc, signature, value); 
}
```

Primeiro verificamos o sinalizador definido no método visitField anterior e chamamos o método visitField novamente, desta vez fornecendo o nome, o modificador de acesso e a descrição. Este método retorna uma instância de FieldVisitor.

O método visitEnd é o último método chamado na ordem dos métodos de visitante. Esta é a posição recomendada para realizar a lógica de inserção do campo.

Em seguida, precisamos chamar o método visitEnd neste objeto para sinalizar que terminamos de visitar este campo:

```
@Override
public void visitEnd() {
    if (!isFieldPresent) {
        FieldVisitor fv = cv.visitField(
          access, fieldName, fieldType, null, null);
        if (fv != null) {
            fv.visitEnd();
        }
    }
    cv.visitEnd();
}
```

É importante ter certeza de que todos os componentes ASM usados vêm do pacote org.objectweb.asm - muitas bibliotecas usam a biblioteca ASM internamente e os IDEs podem inserir automaticamente as bibliotecas ASM agrupadas.

Agora usamos nosso adaptador no método addField, obtendo uma versão transformada de java.lang.Integer com nosso campo adicionado:

```
public class CustomClassWriter {
    AddFieldAdapter addFieldAdapter;
    //...
    public byte[] addField() {
        addFieldAdapter = new AddFieldAdapter(
          "aNewBooleanField",
          org.objectweb.asm.Opcodes.ACC_PUBLIC,
          writer);
        reader.accept(addFieldAdapter, 0);
        return writer.toByteArray();
    }
}
```

Substituímos os métodos visitField e visitEnd.

Tudo a ser feito em relação aos campos acontece com o método visitField. Isso significa que também podemos modificar os campos existentes (digamos, transformar um campo privado em público), alterando os valores desejados passados para o método visitField.


### 4.2. Trabalho com métodos
A geração de métodos inteiros na API ASM envolve mais do que outras operações da classe. Isso envolve uma quantidade significativa de manipulação de código de bytes de baixo nível e, como resultado, está além do escopo deste artigo.

Para a maioria dos usos práticos, entretanto, podemos modificar um método existente para torná-lo mais acessível (talvez torná-lo público para que possa ser substituído ou sobrecarregado) ou modificar uma classe para torná-lo extensível.

Vamos tornar o método toUnsignedString público:

```
public class PublicizeMethodAdapter extends ClassVisitor {
    public PublicizeMethodAdapter(int api, ClassVisitor cv) {
        super(ASM4, cv);
        this.cv = cv;
    }
    public MethodVisitor visitMethod(
      int access,
      String name,
      String desc,
      String signature,
      String[] exceptions) {
        if (name.equals("toUnsignedString0")) {
            return cv.visitMethod(
              ACC_PUBLIC + ACC_STATIC,
              name,
              desc,
              signature,
              exceptions);
        }
        return cv.visitMethod(
          access, name, desc, signature, exceptions);
   }
}
```

Como fizemos para a modificação do campo, meramente interceptamos o método de visita e alteramos os parâmetros que desejamos.

Nesse caso, usamos os modificadores de acesso no pacote org.objectweb.asm.Opcodes para alterar a visibilidade do método. Em seguida, conectamos nosso ClassVisitor:

```
public byte[] publicizeMethod() {
    pubMethAdapter = new PublicizeMethodAdapter(writer);
    reader.accept(pubMethAdapter, 0);
    return writer.toByteArray();
}
```

### 4.3. Trabalho com aulas
Na mesma linha dos métodos de modificação, modificamos as classes interceptando o método de visitante apropriado. Nesse caso, interceptamos a visita, que é o primeiro método na hierarquia do visitante:

```
public class AddInterfaceAdapter extends ClassVisitor {

    public AddInterfaceAdapter(ClassVisitor cv) {
        super(ASM4, cv);
    }

    @Override
    public void visit(
      int version,
      int access,
      String name,
      String signature,
      String superName, String[] interfaces) {
        String[] holding = new String[interfaces.length + 1];
        holding[holding.length - 1] = cloneableInterface;
        System.arraycopy(interfaces, 0, holding, 0, interfaces.length);
        cv.visit(V1_8, access, name, signature, superName, holding);
    }
}
```

Substituímos o método de visita para adicionar a interface Cloneable ao array de interfaces a serem suportadas pela classe Integer. Nós conectamos isso como todos os outros usos de nossos adaptadores.

# 5. Usando a Classe Modificada
Portanto, modificamos a classe Integer. Agora precisamos carregar e usar a versão modificada da classe.

Além de simplesmente gravar a saída de writer.toByteArray no disco como um arquivo de classe, existem algumas outras maneiras de interagir com nossa classe Integer personalizada.

5.1. Usando o TraceClassVisitor
A biblioteca ASM fornece a classe de utilitário TraceClassVisitor que usaremos para fazer uma introspecção da classe modificada. Assim, podemos confirmar que nossas mudanças aconteceram.

Como o TraceClassVisitor é um ClassVisitor, podemos usá-lo como um substituto imediato para um ClassVisitor padrão:

```
PrintWriter pw = new PrintWriter(System.out);

public PublicizeMethodAdapter(ClassVisitor cv) {
    super(ASM4, cv);
    this.cv = cv;
    tracer = new TraceClassVisitor(cv,pw);
}

public MethodVisitor visitMethod(
  int access,
  String name,
  String desc,
  String signature,
  String[] exceptions) {
    if (name.equals("toUnsignedString0")) {
        System.out.println("Visiting unsigned method");
        return tracer.visitMethod(
          ACC_PUBLIC + ACC_STATIC, name, desc, signature, exceptions);
    }
    return tracer.visitMethod(
      access, name, desc, signature, exceptions);
}

public void visitEnd(){
    tracer.visitEnd();
    System.out.println(tracer.p.getText());
}
```

Todas as visitas agora serão feitas com nosso rastreador, que pode imprimir o conteúdo da classe transformada, mostrando todas as modificações que fizemos nela.

Embora a documentação do ASM afirme que o TraceClassVisitor pode imprimir no PrintWriter fornecido ao construtor, isso não parece funcionar corretamente na versão mais recente do ASM.

Felizmente, temos acesso à impressora subjacente na classe e fomos capazes de imprimir manualmente o conteúdo de texto do rastreador em nosso método visitEnd substituído.

### 5.2 Usando instrumentação Java
Esta é uma solução mais elegante que nos permite trabalhar com a JVM em um nível mais próximo via Instrumentação.

Para instrumentar a classe java.lang.Integer, escrevemos um agente que será configurado como um parâmetro de linha de comando com a JVM. O agente requer dois componentes:

- Uma classe que implementa um método denominado premain;
- Uma implementação de ClassFileTransformer na qual forneceremos condicionalmente a versão modificada de nossa classe.

```
public class Premain {
    public static void premain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new ClassFileTransformer() {
            @Override
            public byte[] transform(
              ClassLoader l,
              String name,
              Class c,
              ProtectionDomain d,
              byte[] b)
              throws IllegalClassFormatException {
                if(name.equals("java/lang/Integer")) {
                    CustomClassWriter cr = new CustomClassWriter(b);
                    return cr.addField();
                }
                return b;
            }
        });
    }
}
```

Agora definimos nossa classe de implementação premain em um arquivo de manifesto JAR usando o plugin Maven jar:

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>2.4</version>
    <configuration>
        <archive>
            <manifestEntries>
                <Premain-Class>
                    com.baeldung.examples.asm.instrumentation.Premain
                </Premain-Class>
                <Can-Retransform-Classes>
                    true
                </Can-Retransform-Classes>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```

Construir e empacotar nosso código até agora produz o jar que podemos carregar como um agente. Para usar nossa classe Integer personalizada em uma hipotética “YourClass.class“:

```
java YourClass -javaagent:"/path/to/theAgentJar.jar"
```

# 6. Conclusão
Embora implementemos nossas transformações aqui individualmente, o ASM nos permite encadear vários adaptadores para obter transformações complexas de classes.

Além das transformações básicas que examinamos aqui, o ASM também oferece suporte a interações com anotações, genéricos e classes internas.