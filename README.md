#MVP Helpers

Aftter digging a lot of time with Apps under **MVP pattern**, I started to see classes that could be abstracted in a simpler way in order to use them to build Apps more faster.
This library is intended to provide helpers classes, I don't want to build a robust library because what I love from Android, is the ability to use the patterns you want to create a determinated set of apps.

If you are using MVP, this set of classes could help you a little bit. 

##Installation 

Actually I don't have this library in **JCenter/Maven Central**, so if you want to use, follow this instructions to get everything work: 

**Gradle Installation**

- Add it in your root build.gradle at the end of repositories:
```gradle
allprojects {
	repositories {
		...
		maven { url "https://jitpack.io" }
	}
}
```

- Add the dependency:
```gradle
dependencies {
	 compile 'com.github.BlackBoxVision:mvp-helpers:v0.0.2'
}
```

**Maven Installation**

- Add this line to repositories section in pom.xml:
```xml
<repositories>
	<repository>
	   <id>jitpack.io</id>
		 <url>https://jitpack.io</url>
	</repository>
</repositories>
```
- Add the dependency:
```xml
<dependency>
  <groupId>com.github.BlackBoxVision</groupId>
  <artifactId>mvp-helpers</artifactId>
	<version>v0.0.2</version>
</dependency>
```

**SBT Installation**

- Add it in your build.sbt at the end of resolvers:
```sbt
  resolvers += "jitpack" at "https://jitpack.io"
```

- Add the dependency:
```sbt
  libraryDependencies += "com.github.BlackBoxVision" % "mvp-helpers" % "v0.0.2"	
```

##Usage

The usage is really simple, the concepts behind this library are the following ones: 

- **View** → The **View** is an interface that contains methods related to UI interaction, to create you own you should extends the **BaseView** like in the following example: 

```java
public interface DetailsView extends BaseView {

  void onInfoReceived(@NonNull Bundle information);
  
  void onInfoError(@NonNull String errorMessage);
}
```

- **Interactor** → The **Interactor** is the class that do the hard work, all the blocking operations like **I/O, Networking, Database** Intectations should be done here. 

When you extend the **BaseInteractor** class, you get two useful methods **runOnBackground** and **runOnUiThread**. Both methods receives a **Runnable** as param. 

- **runOnBackground** is a method that executes the runnable you pass to it in an **Executor**. The executor is an instance generated by **Executors** class, it provides you with a fixed thread pool executor of 5 theads. Enough to do hard work. 

- **runOnUiThread** is a method that executes the runnable you pass to it in an **Handler**. The handler instance is generated with a reference to the **Main Looper**. In this way, we now that we really comunicate to the correct thread the updates to post, in our case, the UI one. 

Check the following example: 

```java
//This example uses Java 8 features, I assume the usage of retrolambda
public final class DetailsInteractor extends BaseInteractor {

  public void retrieveDetailsFromService(@NonNull final String id, @NonNull final OnSuccessListener<Bundle> successListener, @NonNull final OnErrorListener<String> errorListener) {
    runOnBackground(() -> {
      //Getting data from somewhere
      Bundle data = ... ;   
      
      if (data != null) {
        runOnUiThread(() -> successListener.onSuccess(data));  
      } else {
        runOnUiThread(() -> errorListener.onError("Ups, something went wrong"));
      }
    })
  }
}
```

- **Presenter** → The **presenter** acts as a middle man between the **Interactor** and the **View**. When you request something to the presenter, he contacts the interactor object to get what he needs, and then he interacts with the view to make you get what you really need. Continue with the example of a Details: 

```java
//I use method references from Java 8 to point the callbacks to interactor, I assume a working project with Retrolambda
public final class DetailsPresenter extends BasePresenter<DetailsView> {
  private DetailsInteractor interactor;
  
  public DetailsPresenter() { 
    interactor = new DetailsInteractor();
  }
  
  public void getInformationFromId(@NonNull String id) {
    if (isViewAttached()) {
      interactor.retrieveDetailsFromService(id, this::onSuccess, this::onError);
    }
  }
  
  private void onSuccess(@NonNull Bundle data) {
    if (isViewAttached()) {
      getView().onInfoReceived(data);
    }
  }
  
  private void onError(@NonNull String errorMessage) {
    if (isViewAttached()) {
      getView().onInfoError(errorMessage);
    }
  }
}
```
As you see, **BasePresenter** is a generic class, you have to pass to it the **View** that you want to use when you inherit from it. **BasePresenter** provides you with a set of methods to deal with **view** interfaces, that are the following ones: 

- **isViewAttached** → This method checks if you have set the view to the presenter, returns to you a boolean value that you should handle in your presenter implementation. 

- **detachView** → This method dereference the view, setting it to null. This method should be called in the onDestroy method in case of use in Activity, and onDestroyView in case of Fragment usage. 

- **registerView** → This method adds the view to the presenter, so you can start to handle the cicle of view - presenter - interactor interaction.

- **getView** → simple getter, to make your access to the view defined more cleaner.

## Complement with Android 

Well, that's the basics behind the library. At this point, you are asking yourself, how do I connect this classes with a Android??. Well, that's pretty simple! 

I work a lot with **Fragments**, they simplify a lot my work flow. I think in them as the **View** in Android. I let Activity manage Fragments, don't want to charge them, since they have a lot of responsabilities. 

This Fragment is a **generic** class that solves some troubles for you, **It has the ability to detach view for you in onDestroyView so you don't have to, auto inject views with butter knife, and lifecycle simplified**. 

When you inherit it, you will get the following methods to implement:

- **addPresenter** → in this method you have to create you instance of Presenter. 

- **getLayout** → in this method you have pass the id reference to the layout. This library comes with **ButterKnife**, to provide efficiency I have implemented **onCreateView** in BaseFragment where I call **ButterKnife.bind** method, so you have view binding out of the box! :smile:

- **getPresenter** → simple getter, to make your access to the presenter more cleaner.

To finalize the explanation, check the sample implementation: 

```java
public final class DetailsFragment extends BaseFragment<DetailsPresenter> implements DetailsView {
    @Override
    public addPresenter() {
      return new DetailsPresenter();
    }
    
    @LayoutRes
    @Override
    public int getLayout() {
      return R.layout.fragment_details;
    }
    
    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        getPresenter.registerView(this);
        getPresenter.getInformationFromId("ssdWRGD132");
    }
    
    @Override
    void onInfoReceived(@NonNull Bundle information) {
       Toast.makeText(getContext(), information.toString(), Toast.LENGTH_SHORT).show();
    }
    
    @Override
    void onInfoError(@NonNull String errorMessage) {
       Toast.makeText(getContext(), errorMessage, Toast.LENGTH_SHORT).show();
    }
} 
```

#License

This library is distributed under **MIT License**.

>The MIT License (MIT)
>Copyright (c) 2016 BlackBox Vision

>Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation >files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, >modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the >Software is furnished to do so, subject to the following conditions:

>The above copyright notice and this permission notice shall be included in all copies or substantial portions of the >Software.

>THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE >WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR >COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, >ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

