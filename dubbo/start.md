# dubbo 笔记

## dubbo xml 元素解析

dubbo 中自定义了对于 dubbo 命名空间的 xml 元素 bean 解析。根据不同的 elementName 名称进行需要解析的 bean 属性对象初始化，用于区分属性值的处理所有的属性值全部放入 BeanDefinetion 的 PropertyValues 中。

dubbo 创建的 Spring bean 对象类型都是同一个命名空间的类型

## dubbo 服务的暴露

dubbo 的 DubboBootstrapApplicationListener 间接实现了 Spring 的 ApplicationListener 接口，Spring 在初始化完成之后会发起一个 event 事件。dubbo 在接收到该事件是进行配置服务的暴露

- DubboBootstrap

暴露服务的启动类，所有的服务暴露都是由该类进行处理
