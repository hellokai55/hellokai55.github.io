---
---

# Booster库剖析一-整体架构分析

# 一、介绍

现在进行Android开发的过程中，免不了需要对部分代码进行hook操作，比如

- 系统bug修复，大家都熟悉的Toast问题
- 隐私合规问题检测
- 统一线程优化等
- ……

我们有了整个APP构建的操作权限，那就可以做很多事情了。但是我们如果直接使用gradle原生API来进行上述优化操作，就会发现一些困扰

1. 代码变化大，上一个版本还是最佳实践下一个版本就要修改了
2. 小团队没有那么大的经历来一直做gradle相关的操作和升级
3. gradle API使用复杂

那就需要一个最好可以开箱即用的，并且API稳定的框架来隔离这些Gradle相关操作，让我们使用稳定的API来对APP构建进行操作，那它就是[Booster](https://booster.johnsonlee.io/zh/)了。

它的优点可以直接看Booster文档介绍，里面有详细的介绍了，就不赘述了，使用起来很简单。

那我们就直接来看看它是如何架构设计的吧！

# 二、架构剖析

在Booster中的[架构剖析](https://booster.johnsonlee.io/zh/architecture/overview.html)中，就有作者对于Booster整体架构设计的理解，可以看出作者的思考很全面，那我们就结合他的文章和源码来看看究竟是怎么写的吧！

文章中也写了其实主要分为四大部分

- 如何实现可扩展？
- 如何让开发变得容易？
- 如何保证构建性能？
- 如何解决开源协议冲突？

那我们就按照这个四楼分别看下代码里是怎么做的吧？

## 2.1 如何实现可扩展？

我们平常在写Gradle的时，需要自定义的也就只有两部分，一个是任务，一个是插件，看看booster如何提供的吧，主要提供了两类的SPI：

- Task SPI
- Transformer SPI

### 2.1.1 Task SPI实现

SPI是啥大家都很熟悉了，就不具体介绍了，主要看他的代码相关是如何实现的吧

```kotlin
package com.didiglobal.booster.task.spi

import com.android.build.gradle.api.BaseVariant

interface VariantProcessor {

    fun process(variant: BaseVariant)
}
```

对于Task任务的抽象，只需要`BaseVariant` 这一个参数即可，提供了19个对应的实现，如果想要自己开发任务相关的继承此接口即可

![Untitled](Booster%E5%BA%93%E5%89%96%E6%9E%90%E4%B8%80-%E6%95%B4%E4%BD%93%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90%2091879783771e42f8b672bdf48aba5661/Untitled.png)

咱们来看一个简单一点的实现类

```kotlin
package com.didiglobal.booster.task.dependency

import com.android.build.gradle.api.BaseVariant
import com.android.build.gradle.internal.tasks.factory.dependsOn
import com.didiglobal.booster.BOOSTER
import com.didiglobal.booster.gradle.javaCompilerTaskProvider
import com.didiglobal.booster.gradle.project
import com.didiglobal.booster.task.spi.VariantProcessor
import com.google.auto.service.AutoService
import org.gradle.api.UnknownTaskException

private const val TASK_NAME = "checkSnapshot"

@AutoService(VariantProcessor::class)
class CheckSnapshotVariantProcessor : VariantProcessor {

    override fun process(variant: BaseVariant) {
        variant.project.tasks.let { tasks ->
            val checkSnapshot = try {
                tasks.named(TASK_NAME)
            } catch (e: UnknownTaskException) {
                tasks.register(TASK_NAME) {
                    it.group = BOOSTER
                    it.description = "Check snapshot dependencies"
                }
            }
            @Suppress("DEPRECATION")
            tasks.register("check${variant.name.capitalize()}Snapshot", CheckSnapshot::class.java) { // <-- 1. 创建一个的任务，任务实现类是CheckSnapshot
                it.group = BOOSTER
                it.description = "Check snapshot dependencies for ${variant.name}"
                it.variant = variant
                it.outputs.upToDateWhen { false }
            }.also {
                variant.javaCompilerTaskProvider.dependsOn(it)
                checkSnapshot.dependsOn(it)
            }
        }
    }

}

//-------------CheckSnapshot 实现类-----------
package com.didiglobal.booster.task.dependency

import com.android.build.gradle.api.BaseVariant
import com.didiglobal.booster.gradle.dependencies
import com.didiglobal.booster.kotlinx.CSI_RESET
import com.didiglobal.booster.kotlinx.CSI_YELLOW
import com.didiglobal.booster.kotlinx.ifNotEmpty
import org.gradle.api.DefaultTask
import org.gradle.api.internal.artifacts.repositories.resolver.MavenUniqueSnapshotComponentIdentifier
import org.gradle.api.tasks.Internal
import org.gradle.api.tasks.TaskAction

internal open class CheckSnapshot : DefaultTask() {

    @get:Internal
    lateinit var variant: BaseVariant

    @TaskAction
    fun run() {
        if (!variant.buildType.isDebuggable) {
            variant.dependencies.filter {
                it.id.componentIdentifier is MavenUniqueSnapshotComponentIdentifier
            }.map {
                it.id.componentIdentifier as MavenUniqueSnapshotComponentIdentifier
            }.ifNotEmpty { snapshots ->
								// <-- 2. 对于带有Snapshot的版本任务的日志输出
                println("$CSI_YELLOW ⚠️  ${snapshots.size} SNAPSHOT artifacts found in ${variant.name} variant:$CSI_RESET\n${snapshots.joinToString("\n") { snapshot -> "$CSI_YELLOW→  ${snapshot.displayName}$CSI_RESET" }}")
            }
        }
    }
}
```

从上述代码我们就看到了它的定义和具体实现是如何设计的，结构清晰，那如何把这些绑定到对应的AGP中的呢，等后面的Transformer代码看完之后再统一说。

### 2.1.2 Transformer SPI实现

直接看他的定义：

```kotlin
package com.didiglobal.booster.transform

/**
 * Represents bytecode transformer
 *
 * @author johnsonlee
 */
interface Transformer : TransformListener {

    /**
     * Returns the transformed bytecode
     *
     * @param context
     *         The transforming context
     * @param bytecode
     *         The bytecode to be transformed
     * @return the transformed bytecode
     */
    fun transform(context: TransformContext, bytecode: ByteArray): ByteArray

}

```

再看看他的实现，比较主要的一个子类就是`AsmTransformer`，以下都分析ASM相关的实现而非`Javassist`。

他是通过一个SPI方式来动态引入的，那如果要开发的话推荐的是使用`ClassTransformer`,那`AsmTransformer` 是如何获取的`ClassTransformer`呢，答案仍然是SPI，那我们先看看`AsmTransformer` 是如何来添加所有的自定义的Transformer呢，它又是如何抽象隔离的呢，我们继续看看源码：

```kotlin
@AutoService(Transformer::class)
class AsmTransformer : Transformer {

    private val threadMxBean = ManagementFactory.getThreadMXBean()

    private val durations = mutableMapOf<ClassTransformer, Duration>()

    private val classLoader: ClassLoader

    internal val transformers: Iterable<ClassTransformer>

    constructor() : this(Thread.currentThread().contextClassLoader)

		// <---1. 直接通过ServiceLoader，来获取ClassTransformer所有的实现类，并通过优先级排序
    constructor(classLoader: ClassLoader = Thread.currentThread().contextClassLoader) : this(ServiceLoader.load(ClassTransformer::class.java, classLoader).sortedBy {
        it.javaClass.getAnnotation(Priority::class.java)?.value ?: 0
    }, classLoader)

    constructor(transformers: Iterable<ClassTransformer>, classLoader: ClassLoader = Thread.currentThread().contextClassLoader) {
        this.classLoader = classLoader
        this.transformers = transformers
    }

		...

    override fun transform(context: TransformContext, bytecode: ByteArray): ByteArray {
        val diffEnabled = context.getProperty("booster.transform.diff", false)
        // <---2. 直接通过ASM库中的ClassWriter，来操作字节码
        return ClassWriter(ClassWriter.COMPUTE_MAXS).also { writer ->
						// <---3. 通过fold来组合所有的ClassTransformer，先转换成ClassNode，后执行transform操作
            this.transformers.fold(ClassNode().also { klass ->
                ClassReader(bytecode).accept(klass, 0)
            }) { a, transformer ->
                this.threadMxBean.sumCpuTime(transformer) {
                    if (diffEnabled) {
                        val left = a.textify()
                        transformer.transform(context, a).also trans@{ b ->
                            val right = b.textify()
                            val diff = if (left == right) "" else left diff right
                            if (diff.isEmpty() || diff.isBlank()) {
                                return@trans
                            }
                            transformer.getReport(context, "${a.className}.diff").touch().writeText(diff)
                        }
                    } else {
                        // <---4.真正修改自己字节码的地方，传递给下面的是ClassNode
                        transformer.transform(context, a)
                    }
                }
            }.accept(writer)
        }.toByteArray()
    }
    ....
}
```

自定的trasform是如何绑定到AsmTransformer中是怎么做的我们也都清楚了，那就需要再回过头来看SPI是怎么绑定的VariantProcessor和AsmTransformer的呢？

### 2.1.3  SPI具体实现

我们先看VariantProcessor的SPI相关代码

```kotlin
package com.didiglobal.booster.gradle

import com.android.build.gradle.AppExtension
import com.android.build.gradle.BaseExtension
import com.android.build.gradle.LibraryExtension
import com.android.build.gradle.api.BaseVariant
import com.didiglobal.booster.task.spi.VariantProcessor
import org.gradle.api.GradleException
import org.gradle.api.Plugin
import org.gradle.api.Project

/**
 * Represents the booster gradle plugin
 *
 * @author johnsonlee
 */
class BoosterPlugin : Plugin<Project> {

    override fun apply(project: Project) {
        project.extensions.findByName("android") ?: throw GradleException("$project is not an Android project")

        if (!GTE_V3_6) {
            project.gradle.addListener(BoosterTransformTaskExecutionListener(project))
        }

        // <---1.主要是这个loadVariantProcessors方法来进行的所有SPI实现的加载
        val processors = loadVariantProcessors(project)
        if (project.state.executed) {
            project.setup(processors)
        } else {
            project.afterEvaluate {
                project.setup(processors)
            }
        }
				// <---2.这个是Transfrom的加载也就是AsmTransformer的加载
        project.getAndroid<BaseExtension>().registerTransform(BoosterTransform.newInstance(project))
    }

    private fun Project.setup(processors: List<VariantProcessor>) {
        val android = project.getAndroid<BaseExtension>()
        when (android) {
            is AppExtension -> android.applicationVariants
            is LibraryExtension -> android.libraryVariants
            else -> emptyList<BaseVariant>()
        }.takeIf<Collection<BaseVariant>>(Collection<BaseVariant>::isNotEmpty)?.let { variants ->
            variants.forEach { variant ->
							  // <---1.6 这里就拿到所有的实现类了然后迭代执行即可
                processors.forEach { processor ->
                    processor.process(variant)
                }
            }
        }
    }
}

// <---1.1 1.0具体调用的方法
@Throws(ServiceConfigurationError::class)
internal fun loadVariantProcessors(project: Project): List<VariantProcessor> {
    return newServiceLoader<VariantProcessor>(project.buildscript.classLoader, Project::class.java).load(project)
}

// <---1.2 1.1具体调用的方法
internal inline fun <reified T> newServiceLoader(classLoader: ClassLoader, vararg types: Class<*>): ServiceLoader<T> {
    return ServiceLoaderFactory(classLoader, T::class.java).newServiceLoader(*types)
}

// <---1.3 真正的ServiceLoader的实现
private class ServiceLoaderImpl<T>(
        private val classLoader: ClassLoader,
        private val service: Class<T>,
        private vararg val types: Class<*>
) : ServiceLoader<T> {

    private val name = "META-INF/services/${service.name}"

    override fun load(vararg args: Any): List<T> {
        return lookup<T>().map { provider ->
            try {
                try {
										// <---1.5 创建真正的实现类对象
                    provider.getConstructor(*types).newInstance(*args) as T
                } catch (e: NoSuchMethodException) {
                    provider.getDeclaredConstructor().newInstance() as T
                }
            } catch (e: Throwable) {
                throw ServiceConfigurationError("Provider $provider not found")
            }
        }
    }

    fun <T> lookup(): Set<Class<T>> {
				// <---1.4 找到通过AutoService库的接口和实现的对应文件，返回一个实现类的集合
        return classLoader.getResources(name)?.asSequence()?.map(::parse)?.flatten()?.toSet()?.mapNotNull { provider ->
            try {
                val providerClass = Class.forName(provider, false, classLoader)
                if (!service.isAssignableFrom(providerClass)) {
                    throw ServiceConfigurationError("Provider $provider not a subtype")
                }
                providerClass as Class<T>
            } catch (e: Throwable) {
                null
            }
        }?.toSet() ?: emptySet()
    }
}
```

从代码里跟1.1到1.6就是VariantProcessor的具体实现逻辑了。

那我们再来看看AsmTransformer相关的吧

入口是在

```kotlin
/**
 * Represents the booster gradle plugin
 *
 * @author johnsonlee
 */
class BoosterPlugin : Plugin<Project> {

    override fun apply(project: Project) {
        project.extensions.findByName("android") ?: throw GradleException("$project is not an Android project")

        if (!GTE_V3_6) {
            project.gradle.addListener(BoosterTransformTaskExecutionListener(project))
        }

        val processors = loadVariantProcessors(project)
        if (project.state.executed) {
            project.setup(processors)
        } else {
            project.afterEvaluate {
                project.setup(processors)
            }
        }
				// <---2.入口：这个是Transfrom的加载也就是AsmTransformer的加载
        project.getAndroid<BaseExtension>().registerTransform(BoosterTransform.newInstance(project))
    }

    private fun Project.setup(processors: List<VariantProcessor>) {
        val android = project.getAndroid<BaseExtension>()
        when (android) {
            is AppExtension -> android.applicationVariants
            is LibraryExtension -> android.libraryVariants
            else -> emptyList<BaseVariant>()
        }.takeIf<Collection<BaseVariant>>(Collection<BaseVariant>::isNotEmpty)?.let { variants ->
            variants.forEach { variant ->
							  // <---1.6 这里就拿到所有的实现类了然后迭代执行即可
                processors.forEach { processor ->
                    processor.process(variant)
                }
            }
        }
    }
}

//BoosterTransform.kt
open class BoosterTransform protected constructor(
        internal val parameter: TransformParameter
) : Transform() {	//<---2.2 真正实现了AGP中的Transform类，也就是真正的和AGP交互的地方

    internal val verifyEnabled by lazy {
        parameter.properties[OPT_TRANSFORM_VERIFY]?.toString()?.toBoolean() ?: false
    }

    override fun getName() = parameter.name

    override fun isIncremental() = !verifyEnabled

    override fun isCacheable() = !verifyEnabled

    override fun getInputTypes(): MutableSet<QualifiedContent.ContentType> = TransformManager.CONTENT_CLASS

    override fun getScopes(): MutableSet<in QualifiedContent.Scope> = when {
        parameter.transformers.isEmpty() -> mutableSetOf()
        parameter.plugins.hasPlugin("com.android.library") -> SCOPE_PROJECT
        parameter.plugins.hasPlugin("com.android.application") -> SCOPE_FULL_PROJECT
        parameter.plugins.hasPlugin("com.android.dynamic-feature") -> SCOPE_FULL_WITH_FEATURES
        else -> TODO("Not an Android project")
    }

    override fun getReferencedScopes(): MutableSet<in QualifiedContent.Scope> = when {
        parameter.transformers.isEmpty() -> when {
            parameter.plugins.hasPlugin("com.android.library") -> SCOPE_PROJECT
            parameter.plugins.hasPlugin("com.android.application") -> SCOPE_FULL_PROJECT
            parameter.plugins.hasPlugin("com.android.dynamic-feature") -> SCOPE_FULL_WITH_FEATURES
            else -> TODO("Not an Android project")
        }
        else -> super.getReferencedScopes()
    }

    final override fun transform(invocation: TransformInvocation) {
				//<---2.3 编译文件处理在这里
        BoosterTransformInvocation(invocation, this).apply {
            if (isIncremental) {
                doIncrementalTransform()
            } else {
                outputProvider?.deleteAll()
                doFullTransform()
            }
        }
    }

    companion object {

        fun newInstance(project: Project, name: String = "booster"): BoosterTransform {
            val parameter = project.newTransformParameter(name)
            return when {
                GTE_V3_4 -> BoosterTransformV34(parameter)
								//<---2.1 创建类的入口
                else -> BoosterTransform(parameter)
            }
        }
    }
}

// BoosterTransformInvocation.kt
// <---2.4 真正的transform地方，看看它怎么做的呢
private fun doTransform(block: (ExecutorService, Set<File>) -> Iterable<Future<*>>) {
  this.outputs.clear()
  this.collectors.clear()

  // 创建线程池，并发执行
  val executor = Executors.newFixedThreadPool(NCPU)

  this.onPreTransform()

  // Look ahead to determine which input need to be transformed even incremental build
  // 这是一个特殊的点可以支持多轮执行的配置，例如一些router需要先统计所有的注册表，再进行字节码操作就需要这个了
  val outOfDate = this.lookAhead(executor).onEach {
      project.logger.info("✨ ${it.canonicalPath} OUT-OF-DATE ")
  }

  try {
			// <---2.5 它是传进来的block真正的实现是transformFully
      block(executor, outOfDate).forEach {
          it.get()
      }
  } finally {
      executor.shutdown()
      executor.awaitTermination(1, TimeUnit.HOURS)
  }

  this.onPostTransform()

  if (transform.verifyEnabled) {
      this.doVerify()
  }
}

// <---2.6 针对于jar和dir文件来进行所有的transfrom
private fun transformFully(executor: ExecutorService, @Suppress("UNUSED_PARAMETER") outOfDate: Set<File>) = this.inputs.map {
    it.jarInputs + it.directoryInputs
}.flatten().map { input ->
    executor.submit {
        val format = if (input is DirectoryInput) Format.DIRECTORY else Format.JAR
        outputProvider?.let { provider ->
            input.transform(provider.getContentLocation(input.id, input.contentTypes, input.scopes, format))
        }
    }
}

private fun QualifiedContent.transform(output: File) = this.file.transform(output)

  private fun File.transform(output: File) {
      outputs += output
      project.logger.info("Booster transforming $this => $output")
      this.transform(output) { bytecode ->
          bytecode.transform()
      }
  }

  private fun ByteArray.transform(): ByteArray {
		  // <---2.7 所有的Transformer的实现类在transformers这个集合里，然后让每个集合里的对象来执行transform
      // 也就是我们上面所给的AsmTransformer中的transform方法了，带有所有的实现的接口
      return transformers.fold(this) { bytes, transformer ->
          transformer.transform(this@BoosterTransformInvocation, bytes)
      }
  }
```

这样的分析下来我们就知道是如何构建这个booster的地基的，主要还是通过的是SPI机制来动态发现实现类的方式来实现的。
