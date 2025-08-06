---
title: Learning Depdency Injection By Building a Library (day one)
date: 2025-08-06 12:00:00 +600
categories: [technical]
tags: [java, dependencyInjection]
---

# What is Depdency Injection?

<p>Dependency Injection (DI) is a way to give an object the things it needs from outside instead of the of object creating them by itself.</p>

In this series of articles we are going to build a depdency injection library.

---

First, let’s understand what we are trying to do. We need something that can create objects for us and give them to our application whenever it needs them. We are going to build that "something."

We’ll start by creating a container. This container will store all the objects in one place. Later, our application can simply ask the container for the object it needs instead of creating it on its own.

Now, what’s the best way to store these objects in the container? The easiest choice is to use a HashMap. In this map, the key will be the object’s name, and the value will be the actual object instance.

So our container should look like this

```java
public class Container {
    private final Map<String, Object> components = new HashMap<>();
    private final Map<Class<?>, String> componentNames = new HashMap<>();
}
```
## Add method to add/get objects from Container

### Methods to Add Components to the Container
Now, we need a way to create instances of certain classes and load them into our container. We don’t want to create instances for every single class—only for specific ones.

To handle this, we can use an annotation to mark the classes we want to load. Let’s call this annotation @Component.

Any class annotated with @Component will be automatically added to the container. Later, other classes in our application can simply request these instances from the container instead of creating them manually.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Component {
    String name() default "";
}
```
```java
package org.sahariardev;

import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.util.HashMap;
import java.util.Map;

public class Container {
    private final Map<String, Object> components = new HashMap<>();
    private final Map<Class<?>, String> componentNames = new HashMap<>();

    private void registerComponent(Class<?> clazz) {
        Component componentAnnotation = clazz.getAnnotation(Component.class);
        String componentName = componentAnnotation.name();

        if (componentName.isEmpty()) {
            componentName = clazz.getSimpleName();
        }

        if (components.containsKey(componentName)) {
            throw new RuntimeException("Component with name: " + componentName + " already exists");
        }

        try {
            Constructor<?> constructor = null;
            try {
                constructor = clazz.getDeclaredConstructor();
            } catch (NoSuchMethodException e) {
                throw new RuntimeException(e);
            }

            Object instance = constructor.newInstance();
            components.put(componentName, instance);
            componentNames.put(instance.getClass(), componentName);
        } catch (InstantiationException | IllegalAccessException | InvocationTargetException e) {
            throw new RuntimeException(
                    "Failed to instantiate component " + componentName + " of type " + clazz.getName() + ": "
                            + e.getMessage(),
                    e);
        }
    }
}

```
This is a very simple method for adding components to the container. Next, we need to scan our packages to find all the classes annotated with @Component and call this method for each of them.

In the next tutorial, we’ll enhance this method to add more features. We’ll introduce an @Autowired annotation to automatically inject dependencies into these components.