I'm currently working on a new form based wizard to make configuring our software a snap. It's reusing a lot of existing code, and so I've created a new form bean and delegated to the existing form beans. 

    public class NewDelegatingForm {

      private OldForm oldForm;
      private OtherOldForm otherOldForm;

      public void setField(String value) {
        oldForm.setField(value);
      }

      public void setOtherField(String value) {
        otherOldForm.setOtherField(value);
      }
    }

I'm then including existing jsp snippets in my new wizard page. The problem is that jsps use reflection to set fields. What happens if someone adds a field to OldForm and doesn't update NewDelegatingForm? Runtime Java errors.

You could make OldForm and OtherOldForm implement interfaces that NewDelegatingForm also impliments. But that still doesn't address the issue that a modification to OldForm will need to update the interface for IOldForm. A simple thing to forget.

To solve this problem I added a task to the end of our build. I can specify a parent class to test against and a delegate class to which we want to ensure the parent class is properly delegating. In my build file I simply create a validating task:

    <target name="validate">
      <taskdef name="verifyDelegation" classname="com.uplogix.buildtools.task.VerifyDelegationTask" classpathref="buildtools.classpath"/>
      <verifyDelegation parentClass="com.uplogix.example.form.NewDelegatingForm" delegateClass="com.uplogix.example.form.OldForm" />
      <verifyDelegation parentClass="com.uplogix.example.form.NewDelegatingForm" delegateClass="com.uplogix.example.form.OtherOldForm" />
    </target>

And the code for the task:

        package com.uplogix.buildtools.task;


        import java.lang.reflect.Field;
        import java.lang.reflect.Method;
        import java.lang.reflect.Modifier;
        import java.util.HashSet;
        import java.util.Set;

        import org.apache.tools.ant.BuildException;
        import org.apache.tools.ant.Project;
        import org.apache.tools.ant.Task;

        public class VerifyDelegationTask extends Task {

          private String parentClass;
          private String delegateClass;


          public void execute() {

            log("Validating...");

            if (parentClass == null) {
              throw new BuildException("Attribute 'parentClass' is required.");
            }

            if (delegateClass == null) {
              throw new BuildException("Attribute 'delegateClass' is required.");
            }

            Set<String> errors = null;
            try {
              errors = validateComposition(Class.forName(parentClass), Class.forName(delegateClass));
            } catch (ClassNotFoundException e) {
              throw new BuildException(e);
            }

            if (errors.size() > 0) {
              for (String error : errors) {
                log(error, Project.MSG_ERR);
              }
              throw new BuildException("Invalid composition for class:" + parentClass);
            }

          }


          public void setParentClass(String parentClass) {

            this.parentClass = parentClass;
          }


          public void setDelegateClass(String delegateClass) {

            this.delegateClass = delegateClass;
          }


          private Set<String> validateComposition(Class parentClass, Class delegateClass) {

            Set<String> errors = new HashSet<String>();
            Field[] fields = parentClass.getDeclaredFields();
            for (Field field : fields) {
              Class fieldClazz = field.getType();
              if (delegateClass.isAssignableFrom(field.getType())) {
                Method[] methods = fieldClazz.getMethods();
                for (Method method : methods) {
                  int modifiers = method.getModifiers();
                  if (Modifier.isStatic(modifiers)) {
                    continue;
                  }
                  if (Modifier.isPrivate(modifiers)) {
                    continue;

                  }
                  try {
                    parentClass.getMethod(method.getName(), method.getParameterTypes());
                  } catch (NoSuchMethodException e) {
                    errors.add(String.format("[%s] Missing method %s", parentClass.getName(), method.toGenericString()));
                  }
                }
              }
            }
            return errors;
          }

        }

While I'm using this for form beans, it can be useful anytime you need to hand off to another class and verify that your API matches.






