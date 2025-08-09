---
title: Learning Depdency Injection By Building a Library (day two)
date: 2025-08-06 12:00:00 +600
categories: [technical]
tags: [java, dependencyInjection]
---

In the previous article, we created a basic register method to add object instances to the container.
In this article, we will enhance that functionality. We’ll scan a base package to find all classes annotated with @Component and load them into the container automatically.
Additionally, we will improve the register method to support multiple constructors and parameterized constructors.
We will also introduce a new annotation, @Autowired, which will be used to inject dependencies.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.CONSTRUCTOR})
public @interface Autowired {
}
```

```java
package org.sahariardev;

import java.io.File;
import java.io.IOException;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.net.URL;
import java.util.*;

public class Container {
    private final Map<String, Object> components = new HashMap<>();
    private final Map<Class<?>, String> componentNames = new HashMap<>();

    public void scan(String basePackage) {
        try {
            Set<Class<?>> classes = getClasses(basePackage);
            for (Class<?> clazz : classes) {
                registerComponent(clazz);
            }
        } catch (ClassNotFoundException | IOException e) {
            throw new RuntimeException(e);
        }
    }

    private Set<Class<?>> getClasses(String packageName) throws ClassNotFoundException, IOException {
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();

        if (classLoader == null) {
            throw new ClassNotFoundException("No class loader found");
        }

        String packageDirName = packageName.replace('.', '/');
        Enumeration<URL> resources = classLoader.getResources(packageDirName);
        List<File> dirs = new ArrayList<>();

        while (resources.hasMoreElements()) {
            URL resource = resources.nextElement();
            dirs.add(new File(resource.getFile()));
        }
        Set<Class<?>> classes = new HashSet<>();
        for (File directory : dirs) {
            findClasses(directory, packageName, classes);
        }

        return classes;
    }

    private void findClasses(File directory, String packageName, Set<Class<?>> classes) {
        if (!directory.exists()) {
            return;
        }

        File[] files = directory.listFiles();
        if (files == null) {
            return;
        }

        for (File file : files) {
            if (file.isDirectory()) {
                findClasses(file, packageName + "." + file.getName(), classes);
            } else if (file.getName().endsWith(".class")) {
                try {
                    String className = packageName + "." + file.getName().substring(0, file.getName().length() - 6);
                    Class<?> clazz = Class.forName(className);
                    classes.add(clazz);
                } catch (ClassNotFoundException e) {
                    System.err.println("Class not found: " + packageName + '.' + file.getName().substring(0,
                            file.getName().length() - 6) + "  Error: " + e.getMessage());
                }
            }
        }
    }

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
                for (Constructor<?> c : clazz.getDeclaredConstructors()) {
                    if (c.isAnnotationPresent(Autowired.class)) {
                        constructor = c;
                        break;
                    }
                }

                if (constructor == null) {
                    throw new RuntimeException("No constructor found for class: " + clazz.getName());
                }
            }

            Object instance;

            if (constructor.getParameterCount() == 0) {
                instance = constructor.newInstance();
            } else {
                Class<?>[] parameterTypes = constructor.getParameterTypes();
                Object[] parameters = new Object[parameterTypes.length];
                for (int i = 0; i < parameterTypes.length; i++) {
                    String depdencyName = componentNames.get(parameterTypes[i]);
                    if (depdencyName == null) {
                        throw new RuntimeException("Component with name: " + componentName + " does not exist");
                    }
                    parameters[i] = components.get(depdencyName);
                    if (parameters[i] == null) {
                        throw new RuntimeException("Component with name: " + componentName + " does not exist");
                    }
                }

                instance = constructor.newInstance(parameters);
            }

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
Now that we have all the methods to populate the container, the next step is to implement methods for injecting dependencies where needed.