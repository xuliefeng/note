# JDK 动态代理 Demo

JDK 的动态代理通过接口实现的方式来进行，代理的类和需要被代理的类要有共同的实现接口。实现的代理类最终会实现这些接口，而且继承自 Proxy 类，该类中有业务实现的 InvocationHandler 对象所有的实现都是通过 InvocationHandler 进行传递调用，在 InvocationHandler 中可以做业务的扩展。

```java

public final class $Proxy0 extends Proxy implements Sub {
    private static Method m1;
    private static Method m6;
    private static Method m3;
    private static Method m4;
    private static Method m2;
    private static Method m5;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void run() throws  {
        try {
            super.h.invoke(this, m6, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String hello(String var1, String var2) throws  {
        try {
            return (String)super.h.invoke(this, m3, new Object[]{var1, var2});
        } catch (RuntimeException | Error var4) {
            throw var4;
        } catch (Throwable var5) {
            throw new UndeclaredThrowableException(var5);
        }
    }

    public final String hello() throws  {
        try {
            return (String)super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String hello(String var1) throws  {
        try {
            return (String)super.h.invoke(this, m5, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m6 = Class.forName("org.lfxu.test.proxy.Sub").getMethod("run");
            m3 = Class.forName("org.lfxu.test.proxy.Sub").getMethod("hello", Class.forName("java.lang.String"), Class.forName("java.lang.String"));
            m4 = Class.forName("org.lfxu.test.proxy.Sub").getMethod("hello");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m5 = Class.forName("org.lfxu.test.proxy.Sub").getMethod("hello", Class.forName("java.lang.String"));
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

```
