- 单例模式：对于某些类对象在整个系统过程中只需要一个就可以满足的情况下。为了资源的开销方面和性能考虑在这个生命周期中我们会只创建一个对象来进行服务提供。单例模式的实现分为两种：延迟加载和启动加载，延迟加载推荐使用内部类静态属性实现，如果是通过双层校验的方式来创建对于定义的单例对象需要使用 volatile 的方式来进行修饰,防止乱序执行导致的并发问题。

- 适配器模式：对于统一的接口公开，由于各种原因具体的实现类不能直接通过统一的接口进行调用。在次需要通过一层转换才能使用，这种方式就是适配器模式。  
  使用模块：日志模块

- 装饰者模式：装饰者模式，主要体现的方式为 使用组合替代继承来实现某些功能。装饰器和组件的实现都实现某个统一的业务接口，组件的实现负责具体的单一模块功能。而装饰器中包含了这个组件对象，并且对该组件添加一些装饰功能来进行业务的扩展实现。

- 建造者模式：建造者模式也叫生成者模式，是将一个复杂对象的构建过程和它的表示分离，从而可以使用同样的构建过程创建不同的表示。建造者模式是将一个负责对象的创建过程分割成一个个简单的步骤，用户只需要了解复杂对象的类型和内容，而无需关心复杂对象的具体构建过程，帮组用户屏蔽了复杂对象的内部具体构建构建细节。

- 策略模式：是指有一定行动内容的相对稳定的策略名称
