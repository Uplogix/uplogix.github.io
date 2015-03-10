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

{% gist tthomas48/2cb6a254ac3b16047c23 %}

While I'm using this for form beans, it can be useful anytime you need to hand off to another class and verify that your API matches.






