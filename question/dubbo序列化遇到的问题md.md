# dubbo默认实现反序列化问题 #
我使用的dubbo版本是2.5.3，从dubbo的官方文档上可以查询到，序列化方式，dubbo协议缺省为hessian2。

    public class Animal implements Serializable {
    	private String name;
    	private int age;
    }

    public class Dog extends Animal {
    	pirvate String name;
    	private Color color;
    }

    // 服务提供者
    public interface IDogService {
    	public void sayHello(Dog dog);
    }

如上代码，如果父类有name字段，为private属性，子类也有name字段，为private属性，如果实例化的是子类，那么name字段set值的时候，通过子类的get方法是可以获取到值的。但是在dubbo服务提供者接口中如果传入的是Dog对象，dog对象name字段设置值了，实际上会发现，服务获取到的name字段的值为空。原因是，dubbo在反序列化对象的时候，使用的hessian2，而hessian2是通过一个HashMap实现的，先将该类的属性值做为Key存入Map中，再递归加载改类的父类属性做为Key存入Map中，如果子类和父类的字段名称一样（如上都叫name），那么父类属性会覆盖子类属性，像本例name字段是私有属性，如果实例化的是Dog对象，最后服务获取到的name字段属性值会为空。

源代码可参加com.alibaba.com.caucho.hessian.io.JavaDeserializer类的getFieldMap(Class)方法

    protected HashMap getFieldMap(Class cl)
      {
    HashMap fieldMap = new HashMap();
    
    for (; cl != null; cl = cl.getSuperclass()) {
      Field []fields = cl.getDeclaredFields();
      for (int i = 0; i < fields.length; i++) {
    Field field = fields[i];

如果将dubbo序列化的实现修改为dubbo自己的方式，如（<dubbo:protocol serialization="dubbo"），和上面一样，name字段设值了，get方法可以获取到。从侧面也印证了dubbo默认使用hessian2序列化的一个问题，在我们使用的时候，在子类与父类之间尽量避免使用同名字段。
