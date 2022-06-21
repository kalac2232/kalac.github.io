---
title: Gradle构建流程分析（一）- 初始化与Task创建
date: 2022-01-12 15:15:24
tags: Android
categories: 
- Android
- Gradle
---


*基于AGP3.4.2版本进行分析*

## 获取源码
在build.gradle中添加`implementation 'com.android.tools.build:gradle:3.4.2'`Sync后即可看到源码
## 插件入口
获取到源码后，看/META-INF/gradle-plugins/com.android.application.properties文件内容为
```
implementation-class=com.android.build.gradle.AppPlugin
```
可以看出插件“com.android.application”对应为**AppPlugin.java**，在同级目录下的com.android.library.properties文件
```
implementation-class=com.android.build.gradle.LibraryPlugin
```
可以看出插件“com.android.library”为**LibraryPlugin.java**

<!-- more -->

## 核心逻辑
`AppPlugin`与`LibraryPlugin`中并没有实现`Plugin#apply()`方法，该方法是由其父类`BasePlugin`实现

```java
//BasePlugin#apply()
@Override
public final void apply(@NonNull Project project) {
    CrashReporting.runAction(
            () -> {
            		// AppPlugin与LibraryPlugin即其他插件通用逻辑
                basePluginApply(project);
                // 子类按需添加的功能
                pluginSpecificApply(project);
            });
}
```
`pluginSpecificApply`为一个抽象方法，供子类在`apply()`方法中添加自定义功能。`AppPlugin`中为空实现，在`LibraryPlugin`中添加了一个`assembleDefault`的Task。

```java
//LibraryPlugin#pluginSpecificApply()
@Override
protected void pluginSpecificApply(@NonNull Project project) {
    // Default assemble task for the default-published artifact.
    // This is needed for the prepare task on the consuming project.
    project.getTasks().create("assembleDefault");
}
```

`basePluginApply`方法为核心方法，其中处理了整个插件的初始化工作，Gradle版本、依赖检查、预设build任务完成后的清理工作等。其中最重要的是设置Extension与创建Task

```java
// BasePlugin#basePluginApply()
private void basePluginApply(@NonNull Project project) {
    // ....初始化等工作

  	// 一个目前试验性的参数，目前还没启用
    if (!projectOptions.get(BooleanOption.ENABLE_NEW_DSL_AND_API)) {
				// 创建核心类、下载缺少的sdk等
        threadRecorder.record(
                ExecutionType.BASE_PLUGIN_PROJECT_CONFIGURE,
                project.getPath(),
                null,
                this::configureProject);
				// 创建配置表
        threadRecorder.record(
                ExecutionType.BASE_PLUGIN_PROJECT_BASE_EXTENSION_CREATION,
                project.getPath(),
                null,
                this::configureExtension);
				// 创建Task
        threadRecorder.record(
                ExecutionType.BASE_PLUGIN_PROJECT_TASKS_CREATION,
                project.getPath(),
                null,
                this::createTasks);
    } else {
        // ...
    }
}
```
其中需要重点关注3个方法`configureProject()`,`configureExtension()`,`createTasks()`。

### 配置环境 - configureProject() 

`configureProject`中创建了下文所需的`androidBuilder`、`globalScope`核心类

```java
// BasePlugin#configureProject()
private void configureProject() {
		// ....
    extraModelInfo = new ExtraModelInfo(project.getPath(), projectOptions, project.getLogger());

    sdkHandler = new SdkHandler(project, getLogger());
    // 下载缺少的sdk
    if (!gradle.getStartParameter().isOffline()
            && projectOptions.get(BooleanOption.ENABLE_SDK_DOWNLOAD)) {
        SdkLibData sdkLibData = SdkLibData.download(getDownloader(), getSettingsController());
        sdkHandler.setSdkLibData(sdkLibData);
    }
		// 创建主构建器，合并menifest、合并resources等实现都是由该类完成的
    AndroidBuilder androidBuilder =
            new AndroidBuilder(
                    project == project.getRootProject() ? project.getName() : project.getPath(),
                    creator,
                    new GradleProcessExecutor(project),
                    new GradleJavaProcessExecutor(project),
                    extraModelInfo.getSyncIssueHandler(),
                    extraModelInfo.getMessageReceiver(),
                    getLogger());
    // DataBinding相关
    dataBindingBuilder = new DataBindingBuilder();
    
		// .....

		// 全局的数据存储
    globalScope =
            new GlobalScope(
                    project,
                    new ProjectWrapper(project),
                    projectOptions,
                    dslScope,
                    androidBuilder,
                    sdkHandler,
                    registry,
                    buildCache);

    project.getTasks()
            .getByName("assemble")
            .setDescription(
                    "Assembles all variants of all applications and secondary packages.");

    // call back on execution. This is called after the whole build is done (not
    // after the current project is done).
    // This is will be called for each (android) projects though, so this should support
    // being called 2+ times.
    gradle.addBuildListener(
            new BuildListener() {
                @Override
                public void buildStarted(@NonNull Gradle gradle) {}

                @Override
                public void settingsEvaluated(@NonNull Settings settings) {}

                @Override
                public void projectsLoaded(@NonNull Gradle gradle) {}

                @Override
                public void projectsEvaluated(@NonNull Gradle gradle) {}

                @Override
                public void buildFinished(@NonNull BuildResult buildResult) {
                    // Do not run buildFinished for included project in composite build.
                  	// 清理工作
                    if (buildResult.getGradle().getParent() != null) {
                        return;
                    }
                    ModelBuilder.clearCaches();
                    sdkHandler.unload();
                    threadRecorder.record(
                            ExecutionType.BASE_PLUGIN_BUILD_FINISHED,
                            project.getPath(),
                            null,
                            () -> {
                                WorkerActionServiceRegistry.INSTANCE
                                        .shutdownAllRegisteredServices(
                                                ForkJoinPool.commonPool());
                                Main.clearInternTables();
                            });
                    DeprecationReporterImpl.Companion.clean();
                }
            });

    createLintClasspathConfiguration(project);
}
```

### 创建Extension - configureExtension()

在`configureExtension`中创建了`Extension`，使得我们可以在build.gradle中的使用这样的代码块
```groovy
android{
  buildToolsVersion '28.0.3'
      defaultConfig {
          applicationId "com.xxx"
          versionCode 1157
      }
}

```

`Extension`详解可以参照官方文档：[Gradle Plugin](https://docs.gradle.org/current/userguide/custom_plugins.html)

```java
// BasePlugin#configureExtension()

private void configureExtension() {
    ObjectFactory objectFactory = project.getObjects();
  	// buildTypes
    final NamedDomainObjectContainer<BuildType> buildTypeContainer =
            project.container(
                    BuildType.class,
                    new BuildTypeFactory(
                            objectFactory,
                            project,
                            extraModelInfo.getSyncIssueHandler(),
                            extraModelInfo.getDeprecationReporter()));
  	// productFlavors
    final NamedDomainObjectContainer<ProductFlavor> productFlavorContainer =
            project.container(
                    ProductFlavor.class,
                    new ProductFlavorFactory(
                            objectFactory,
                            project,
                            extraModelInfo.getDeprecationReporter(),
                            project.getLogger()));
  	// signingConfigs
    final NamedDomainObjectContainer<SigningConfig> signingConfigContainer =
            project.container(
                    SigningConfig.class,
                    new SigningConfigFactory(
                            objectFactory,
                            GradleKeystoreHelper.getDefaultDebugKeystoreLocation()));

    final NamedDomainObjectContainer<BaseVariantOutput> buildOutputs =
            project.container(BaseVariantOutput.class);

    project.getExtensions().add("buildOutputs", buildOutputs);
		// sourceSets 可以配置资源的位置，
    sourceSetManager =
            new SourceSetManager(
                    project,
                    isPackagePublished(),
                    globalScope.getDslScope(),
                    new DelayedActionsExecutor());
		// 根据子类创建Extension
    extension =
            createExtension(
                    project,
                    projectOptions,
                    globalScope,
                    sdkHandler,
                    buildTypeContainer,
                    productFlavorContainer,
                    signingConfigContainer,
                    buildOutputs,
                    sourceSetManager,
                    extraModelInfo);
		// 存储到globalScope中
    globalScope.setExtension(extension);

  	// 针对app、lib进行不同的实现
    variantFactory = createVariantFactory(globalScope, extension);
		// 创建TaskManager
    taskManager =
            createTaskManager(
                    globalScope,
                    project,
                    projectOptions,
                    dataBindingBuilder,
                    extension,
                    sdkHandler,
                    variantFactory,
                    registry,
                    threadRecorder);

    variantManager =
            new VariantManager(
                    globalScope,
                    project,
                    projectOptions,
                    extension,
                    variantFactory,
                    taskManager,
                    sourceSetManager,
                    threadRecorder);

    registerModels(registry, globalScope, variantManager, extension, extraModelInfo);

  	// 每一个signingConfigs配置即会调用一次addSigningConfig，使得将所有的签名文件存放到一个map中。
  	// 即使不配置signingConfigs也会有一个默认的DEBUG签名。
    // map the whenObjectAdded callbacks on the containers.
    signingConfigContainer.whenObjectAdded(variantManager::addSigningConfig);

    buildTypeContainer.whenObjectAdded(
            buildType -> {
                if (!this.getClass().isAssignableFrom(DynamicFeaturePlugin.class)) {
                    SigningConfig signingConfig =
                            signingConfigContainer.findByName(BuilderConstants.DEBUG);
                  	// 将项目的默认签名设置为DEBUG签名
                    buildType.init(signingConfig);
                } else {
                    // initialize it without the signingConfig for dynamic-features.
                    buildType.init();
                }
                variantManager.addBuildType(buildType);
            });
		// 每个flavor也会调用addProductFlavor方法，分析每个flavor中独立配置的参数列表，
  	// 包括这个flover独立的minSdkVersion、targetSdkVersion等，构建一个ProductFlavorData数据
    productFlavorContainer.whenObjectAdded(variantManager::addProductFlavor);

    // map whenObjectRemoved on the containers to throw an exception.
    signingConfigContainer.whenObjectRemoved(
            new UnsupportedAction("Removing signingConfigs is not supported."));
    buildTypeContainer.whenObjectRemoved(
            new UnsupportedAction("Removing build types is not supported."));
    productFlavorContainer.whenObjectRemoved(
            new UnsupportedAction("Removing product flavors is not supported."));

  	// 在signingConfig创建一个debug dsl，在buildTypes中创建debug、release的dsl
    // create default Objects, signingConfig first as its used by the BuildTypes.
    variantFactory.createDefaultComponents(
            buildTypeContainer, productFlavorContainer, signingConfigContainer);
}
```

在configureExtension方法中，首先配置了我们经常在build.gradle中看到的一些参数，例如signingConfigs、productFlavors等，并通过`whenObjectAdded`监听将填入的数据收集到variantManager中的map中。在抽象方法`createExtension`中会根据子类的不同进行创建不同的`Extension`。具体都是直接调用`project.getExtensions().create("android",getExtensionClass(),xxxxxx)`方法进行创建，AppPlugin`getExtensionClass()`返回的是`BaseAppModuleExtension.class`，LibraryPlugin方法返回的是`LibraryExtension.class`。

```java
// AppPlugin#getExtensionClass()
@Override
@NonNull
protected Class<? extends AppExtension> getExtensionClass() {
    return BaseAppModuleExtension.class;
}
// LibraryPlugin#getExtensionClass()
@NonNull
protected Class<? extends BaseExtension> getExtensionClass() {
    return LibraryExtension.class;
}
```

同时，也创建了`createTasks`方法中所需的参数：taskManager、variantManager。

### 创建Task列表-createTasks() 


```java
private void createTasks() {
    threadRecorder.record(
            ExecutionType.TASK_MANAGER_CREATE_TASKS,
            project.getPath(),
            null,
            () -> taskManager.createTasksBeforeEvaluate()); // 创建一些不依赖flover的Task

    project.afterEvaluate(
            CrashReporting.afterEvaluate(
                    p -> {
                      	// 执行延时任务，一般为空任务
                        sourceSetManager.runBuildableArtifactsActions();

                        threadRecorder.record(
                                ExecutionType.BASE_PLUGIN_CREATE_ANDROID_TASKS,
                                project.getPath(),
                                null,
                                this::createAndroidTasks);// 开始创建核心Task
                    }));
}
```

在`taskManager.createTasksBeforeEvaluate()`中，创建了`preBuild`、lint相关的task。构建核心task是在createAndroidTasks方法中创建的。

```java

@VisibleForTesting
final void createAndroidTasks() {
    // ... 参数的检查与设置

  	// lint 
    taskManager.configureCustomLintChecks();
  
		// ...

  	// 创建Task的核心方法，会对buildTypes 和 flover进行组合，并创建合并后tasks
    List<VariantScope> variantScopes = variantManager.createAndroidTasks();

    ApiObjectFactory apiObjectFactory =
            new ApiObjectFactory(
                    globalScope.getAndroidBuilder(),
                    extension,
                    variantFactory,
                    project.getObjects());
    for (VariantScope variantScope : variantScopes) {
        BaseVariantData variantData = variantScope.getVariantData();
        apiObjectFactory.create(variantData);
    }

    // Make sure no SourceSets were added through the DSL without being properly configured
    // Only do it if we are not restricting to a single variant (with Instant
    // Run or we can find extra source set
    if (projectOptions.get(StringOption.IDE_RESTRICT_VARIANT_NAME) == null) {
        sourceSetManager.checkForUnconfiguredSourceSets();
    }

    // must run this after scopes are created so that we can configure kotlin
    // kapt tasks
    taskManager.addDataBindingDependenciesIfNecessary(
            extension.getDataBinding(), variantManager.getVariantScopes());


    // create the global lint task that depends on all the variants
    taskManager.configureGlobalLintTask(variantManager.getVariantScopes());

    int flavorDimensionCount = 0;
    if (extension.getFlavorDimensionList() != null) {
        flavorDimensionCount = extension.getFlavorDimensionList().size();
    }

    taskManager.createAnchorAssembleTasks(
            variantScopes,
            extension.getProductFlavors().size(),
            flavorDimensionCount,
            variantFactory.getVariantConfigurationTypes().size());

    // now publish all variant artifacts.
    for (VariantScope variantScope : variantManager.getVariantScopes()) {
        variantManager.publishBuildArtifacts(variantScope);
    }

    checkSplitConfiguration();
    variantManager.setHasCreatedTasks(true);
}
```



```java
public List<VariantScope> createAndroidTasks() {
    variantFactory.validateModel(this);
    variantFactory.preVariantWork(project);

    if (variantScopes.isEmpty()) {
    		// 拼接flover与buildType
        populateVariantDataList();
    }

    // Create top level test tasks.
    taskManager.createTopLevelTestTasks(!productFlavors.isEmpty());

    for (final VariantScope variantScope : variantScopes) {
      	// 根据每个组合创建Task
        createTasksForVariantData(variantScope);
    }

    taskManager.createSourceSetArtifactReportTask(globalScope);

    taskManager.createReportTasks(variantScopes);

    return variantScopes;
}
```

```java
public void createTasksForVariantData(final VariantScope variantScope) {
    final BaseVariantData variantData = variantScope.getVariantData();
    final VariantType variantType = variantData.getType();
    final GradleVariantConfiguration variantConfig = variantScope.getVariantConfiguration();

  	// 与variantData拼接名称创建assembleTask
    taskManager.createAssembleTask(variantData);
    if (variantType.isBaseModule()) {
        taskManager.createBundleTask(variantData);
    }

    if (variantType.isTestComponent()) {
        // ...

    } else {
      	// 创建其他Task
        taskManager.createTasksForVariantScope(variantScope);
    }
}
```

```java
// ApplicationTaskManager#createTasksForVariantScope()
@Override
public void createTasksForVariantScope(@NonNull final VariantScope variantScope) {
    createAnchorTasks(variantScope);
    createCheckManifestTask(variantScope);

    handleMicroApp(variantScope);

    // Create all current streams (dependencies mostly at this point)
    createDependencyStreams(variantScope);

    // Add a task to publish the applicationId.
    createApplicationIdWriterTask(variantScope);

    taskFactory.register(new MainApkListPersistence.CreationAction(variantScope));
    createBuildArtifactReportTask(variantScope);

    // Add a task to process the manifest(s)
    createMergeApkManifestsTask(variantScope);

    // Add a task to create the res values
    createGenerateResValuesTask(variantScope);

    // Add a task to compile renderscript files.
    createRenderscriptTask(variantScope);

    // Add a task to merge the resource folders
    createMergeResourcesTask(
            variantScope,
            true,
            Sets.immutableEnumSet(MergeResources.Flag.PROCESS_VECTOR_DRAWABLES));

    // Add tasks to compile shader
    createShaderTask(variantScope);

    // Add a task to merge the asset folders
    createMergeAssetsTask(variantScope);

    // Add a task to create the BuildConfig class
    createBuildConfigTask(variantScope);

    // Add a task to process the Android Resources and generate source files
    createApkProcessResTask(variantScope);

    // Add a task to process the java resources
    createProcessJavaResTask(variantScope);

    createAidlTask(variantScope);

    // Add external native build tasks
    createExternalNativeBuildJsonGenerators(variantScope);
    createExternalNativeBuildTasks(variantScope);

    // Add a task to merge the jni libs folders
    createMergeJniLibFoldersTasks(variantScope);

    // Add feature related tasks if necessary
    if (variantScope.getType().isBaseModule()) {
        // Base feature specific tasks.
        taskFactory.register(new FeatureSetMetadataWriterTask.CreationAction(variantScope));

        createValidateSigningTask(variantScope);
        // Add a task to produce the signing config file.
        taskFactory.register(new SigningConfigWriterTask.CreationAction(variantScope));

        if (extension.getDataBinding().isEnabled()) {
            // Create a task that will package the manifest ids(the R file packages) of all
            // features into a file. This file's path is passed into the Data Binding annotation
            // processor which uses it to known about all available features.
            //
            // <p>see: {@link TaskManager#setDataBindingAnnotationProcessorParams(VariantScope)}
            taskFactory.register(
                    new DataBindingExportFeatureApplicationIdsTask.CreationAction(
                            variantScope));

        }
    } else {
        // Non-base feature specific task.
        // Task will produce artifacts consumed by the base feature
        taskFactory.register(
                new FeatureSplitDeclarationWriterTask.CreationAction(variantScope));
        if (extension.getDataBinding().isEnabled()) {
            // Create a task that will package necessary information about the feature into a
            // file which is passed into the Data Binding annotation processor.
            taskFactory.register(
                    new DataBindingExportFeatureInfoTask.CreationAction(variantScope));
        }
        taskFactory.register(new MergeConsumerProguardFilesTask.CreationAction(variantScope));
    }

    // Add data binding tasks if enabled
    createDataBindingTasksIfNecessary(variantScope, MergeType.MERGE);

    // Add a compile task
    createCompileTask(variantScope);

    createStripNativeLibraryTask(taskFactory, variantScope);


    if (variantScope.getVariantData().getMultiOutputPolicy().equals(MultiOutputPolicy.SPLITS)) {
        if (extension.getBuildToolsRevision().getMajor() < 21) {
            throw new RuntimeException(
                    "Pure splits can only be used with buildtools 21 and later");
        }

        createSplitTasks(variantScope);
    }


    TaskProvider<BuildInfoWriterTask> buildInfoWriterTask =
            createInstantRunPackagingTasks(variantScope);
    createPackagingTask(variantScope, buildInfoWriterTask);

    // Create the lint tasks, if enabled
    createLintTasks(variantScope);

    taskFactory.register(new FeatureSplitTransitiveDepsWriterTask.CreationAction(variantScope));

    createDynamicBundleTask(variantScope);
}
```

