Node 编译及运行解析
前言
本篇分享以 node 12.11.1 版本为参考，截取了部分关键代码帮助理解，阅读的时候也可以直接看源代码
编译
编译 node 可以参考 Readme 的说明

以下命令可以直接编译 release 版本：
./configure
make -j4

debug 版本也有说明，configure 的时候加上 --debug 即可
./configure --debug
make -j4

是不是感觉很熟悉，没错就是我们一般编译 C++ 项目的步骤

Configure
看一下 configure 做了什么，configure 文件主要是用 py2.7 import configure，这个不多介绍
configure.py
主要看 configure.py，这段 py 代码比较长，大概分为几部分：

- 引入库和初始化 configure 的命令行参数及说明，有兴趣可以看一下 --help 的帮助信息
import xxx
...
parser.add_option(...)
...

- 定义一系列函数
def xxx():
    ...

- 根据配置生成 config.gypi
output = {
  'variables': {},
  'include_dirs': [],
  'libraries': [],
  'defines': [],
  'cflags': [],
}
...
configure_node(output)
configure_napi(output)
configure_library('zlib', output)
configure_library('http_parser', output)
configure_library('libuv', output)
configure_library('libcares', output)
configure_library('nghttp2', output)
# stay backwards compatible with shared cares builds
output['variables']['node_shared_cares'] = \
    output['variables'].pop('node_shared_libcares')
configure_v8(output)
configure_openssl(output)
configure_intl(output)
configure_static(output)
configure_inspector(output)

...
write('config.gypi', do_not_edit +
      pprint.pformat(output, indent=2) + '\n')

- 运行 run_gyp
from gyp_node import run_gyp
...
gyp_args = ['--no-parallel', '-Dconfiguring_node=1']
...
print_verbose("running: \n    " + " ".join(['python', 'tools/gyp_node.py'] + gyp_args))
run_gyp(gyp_args)
再去看一下 tools/gyp_node.py 的代码，可以看到做了以下几件事情：
- 引入了 gyp 的库
- 定义输出为 out 文件夹
- 添加源代码里有的 node.gyp、common.gypi 以及刚才生成的 config.gypi
- 最后运行 gyp.main
...
import gyp

output_dir = os.path.join(os.path.abspath(node_root), 'out')

def run_gyp(args):
  args.append(os.path.join(a_path, 'node.gyp'))
  common_fn = os.path.join(a_path, 'common.gypi')
  options_fn = os.path.join(a_path, 'config.gypi')
  options_fips_fn = os.path.join(a_path, 'config_fips.gypi')

  if os.path.exists(common_fn):
    args.extend(['-I', common_fn])

  if os.path.exists(options_fn):
    args.extend(['-I', options_fn])

  if os.path.exists(options_fips_fn):
    args.extend(['-I', options_fips_fn])

  args.append('--depth=' + node_root)

  # There's a bug with windows which doesn't allow this feature.
  if sys.platform != 'win32' and 'ninja' not in args:
    # Tell gyp to write the Makefiles into output_dir
    args.extend(['--generator-output', output_dir])

    # Tell make to write its output into the same dir
    args.extend(['-Goutput_dir=' + output_dir])

  args.append('-Dcomponent=static_library')
  args.append('-Dlibrary=static_library')

  # Don't compile with -B and -fuse-ld=, we don't bundle ld.gold.  Can't be
  # set in common.gypi due to how deps/v8/build/toolchain.gypi uses them.
  args.append('-Dlinux_use_bundled_binutils=0')
  args.append('-Dlinux_use_bundled_gold=0')
  args.append('-Dlinux_use_gold_flags=0')

  rc = gyp.main(args)
  if rc != 0:
    print('Error running GYP')
    sys.exit(rc)

gyp
（gyp 是 google 做的项目构建工具，可以用来生成 Makefile）
这里我们主要关心 node.gyp 配置的核心部分即可
- variables，定义的各个全部变量名
- targets，编译目标
- includes、include_dirs，包含的头文件
- sources，源文件
- actions，执行命令
在 node.gyp 中主要的 target 是 node 和 libnode，这个先简单提一下，后面部分会做详细介绍
{
  'variables': {
    ...,
    'node_core_target_name%': 'node',
    'node_lib_target_name%': 'libnode',
    'library_files': [
      'lib/internal/bootstrap/environment.js',
      'lib/internal/bootstrap/loaders.js',
      'lib/internal/bootstrap/node.js',
      'lib/internal/bootstrap/pre_execution.js',
      ...,
    ],
  },
  'targets': [
    {
      'target_name': '<(node_core_target_name)',
      'type': 'executable',
      ...,
      'include_dirs': [
        'src',
        'deps/v8/include'
      ],
      'sources': [
        'src/node_main.cc'
      ],
      ...
    },
    {
      'target_name': '<(node_lib_target_name)',
      ...,
      'include_dirs': [
        'src',
        '<(SHARED_INTERMEDIATE_DIR)'
      ],
      'sources': [
        'src/api/async_resource.cc',
        'src/api/callback.cc',
        'src/api/encoding.cc',
        'src/api/environment.cc',
        'src/api/exceptions.cc',
        'src/api/hooks.cc',
        'src/api/utils.cc',
        'src/async_wrap.cc',
        ...
      ],
      'actions': [
        {
          'action_name': 'node_js2c',
          'process_outputs_as_sources': 1,
          'inputs': [
            'tools/js2c.py',
            '<@(library_files)',
            ...
          ],
          'outputs': [
            '<(SHARED_INTERMEDIATE_DIR)/node_javascript.cc',
          ],
          'action': [
            'python', '<@(_inputs)', '--target', '<@(_outputs)',
          ],
      ]
    }
  ]
}
Makefile
因为 Makefile 大部分是 gyp 生成的代码，简要看一下
- node 源代码中的 Makefile
ifeq ($(BUILDTYPE),Release)
all: $(NODE_EXE) ## Default target, builds node in out/Release/node.
else
all: $(NODE_EXE) $(NODE_G_EXE)
endif

ifeq ($(BUILD_WITH), make)
$(NODE_EXE): config.gypi out/Makefile
  $(MAKE) -C out BUILDTYPE=Release V=$(V)
  if [ ! -r $@ -o ! -L $@ ]; then ln -fs out/Release/$(NODE_EXE) $@; fi

$(NODE_G_EXE): config.gypi out/Makefile
  $(MAKE) -C out BUILDTYPE=Debug V=$(V)
  if [ ! -r $@ -o ! -L $@ ]; then ln -fs out/Debug/$(NODE_EXE) $@; fi
- 生成的 out/Makefile，引用 out 中生成的其他 mk 文件
ifeq ($(strip $(foreach prefix,$(NO_LOAD),\
    $(findstring $(join ^,$(prefix)),\
                 $(join ^,libnode.target.mk)))),)
  include libnode.target.mk
endif
ifeq ($(strip $(foreach prefix,$(NO_LOAD),\
    $(findstring $(join ^,$(prefix)),\
                 $(join ^,node.target.mk)))),)
  include node.target.mk
endif
...

# "all" is a concatenation of the "all" targets from all the included
# sub-makefiles. This is just here to clarify.
all:

# Add in dependency-tracking rules.  $(all_deps) is the list of every single
# target in our tree. Only consider the ones with .d (dependency) info:
d_files := $(wildcard $(foreach f,$(all_deps),$(depsdir)/$(f).d))
ifneq ($(d_files),)
  include $(d_files)
endif
- 生成的 node.target.mk，用编译的静态链接库生成 node 可执行文件
$(builddir)/node: GYP_LDFLAGS := $(LDFLAGS_$(BUILDTYPE))
$(builddir)/node: LIBS := $(LIBS)
$(builddir)/node: GYP_LIBTOOLFLAGS := $(LIBTOOLFLAGS_$(BUILDTYPE))
$(builddir)/node: LD_INPUTS := $(OBJS) $(builddir)/libhistogram.a $(builddir)/libnode.a $(builddir)/libv8_libplatform.a $(builddir)/libicui18n.a $(builddir)/libzlib.a $(builddir)/libhttp_parser.a $(builddir)/libllhttp.a $(builddir)/libcares.a $(builddir)/libuv.a $(builddir)/libnghttp2.a $(builddir)/libbrotli.a $(builddir)/libopenssl.a $(builddir)/libv8_base_without_compiler.a $(builddir)/libicuucx.a $(builddir)/libicudata.a $(builddir)/libicustubdata.a $(builddir)/libv8_libbase.a $(builddir)/libv8_libsampler.a $(builddir)/libv8_compiler.a $(builddir)/libv8_snapshot.a $(builddir)/libv8_initializers.a
$(builddir)/node: TOOLSET := $(TOOLSET)
$(builddir)/node: $(OBJS) $(builddir)/libhistogram.a $(builddir)/libnode.a $(builddir)/libv8_libplatform.a $(builddir)/libicui18n.a $(builddir)/libzlib.a $(builddir)/libhttp_parser.a $(builddir)/libllhttp.a $(builddir)/libcares.a $(builddir)/libuv.a $(builddir)/libnghttp2.a $(builddir)/libbrotli.a $(builddir)/libopenssl.a $(builddir)/libv8_base_without_compiler.a $(builddir)/libicuucx.a $(builddir)/libicudata.a $(builddir)/libicustubdata.a $(builddir)/libv8_libbase.a $(builddir)/libv8_libsampler.a $(builddir)/libv8_compiler.a $(builddir)/libv8_snapshot.a $(builddir)/libv8_initializers.a FORCE_DO_CMD

all: $(builddir)/node

Make
有了 Makefile 之后，make 的编译过程就不多介绍了
运行
有了前面对编译过程的认识，再来看 node 的运行过程就自然有了出发点

看一下前面 node.gyp 中的配置，可以看到 node 配置的入口文件是 src/node_main.cc，可以看到 main 函数主要是运行了 node::Start
int main(int argc, char* argv[]) {
  ...
  setvbuf(stdout, nullptr, _IONBF, 0);
  setvbuf(stderr, nullptr, _IONBF, 0);
  return node::Start(argc, argv);
}

再找一下 node::Start 的代码，可以看到在 src/node.cc 中
int Start(int argc, char** argv) {
  InitializationResult result = InitializeOncePerProcess(argc, argv);
  ...
  {
    ...
    NodeMainInstance main_instance(&params,
                                   uv_default_loop(),
                                   per_process::v8_platform.Platform(),
                                   result.args,
                                   result.exec_args,
                                   indexes);
    result.exit_code = main_instance.Run();
  }
  TearDownOncePerProcess();
  return result.exit_code;
}
这里主要是两个方法 InitializeOncePerProcess 和 NodeMainInstance，分别看一下
InitializeOncePerProcess
这个方法名可以看出来，每个进程运行的初始化方法，这个相同的方法名会在后面的代码中多次出现，代表的都是一个意思
InitializationResult InitializeOncePerProcess(int argc, char** argv) {
  atexit(ResetStdio);
  PlatformInit();
  ...
  InitializationResult result;
  ...
  InitializationResult result;
  result.args = std::vector<std::string>(argv, argv + argc);
  std::vector<std::string> errors;
  result.exit_code =
       InitializeNodeWithArgs(&(result.args), &(result.exec_args), &errors);
  ...
  InitializeV8Platform(per_process::cli_options->v8_thread_pool_size);
  V8::Initialize();
  performance::performance_v8_start = PERFORMANCE_NOW();
  per_process::v8_initialized = true;
  return result;
InitializeOncePerProcess 一共执行了四个主要的方法
- PlatformInit
- InitializeNodeWithArgs
- InitializeV8Platform
- V8::Initialize
下面分别看一下这四个方法
PlatformInit
主要对各系统做一些初始化操作，不多做介绍

InitializeNodeWithArgs
int InitializeNodeWithArgs(std::vector<std::string>* argv,
                           std::vector<std::string>* exec_argv,
                           std::vector<std::string>* errors) {
  // Initialize node_start_time to get relative uptime.
  per_process::node_start_time = uv_hrtime();

  // Register built-in modules
  binding::RegisterBuiltinModules();

  ...

  const int exit_code = ProcessGlobalArgs(argv, exec_argv, errors, false);
  if (exit_code != 0) return exit_code;

  return 0;
}
可以看到主要有
- 记录时间，用的是 libuv 的 uv_hrtime 
- 注册 C++ 的 builtin 模块，跟踪一下 RegisterBuiltinModules 就可以看到
- 初始化和解析运行参数

InitializeV8Platform
这里主要是初始化 NodePlatform
struct V8Platform {
#if NODE_USE_V8_PLATFORM
  inline void Initialize(int thread_pool_size) {
    ...
    // Tracing must be initialized before platform threads are created.
    platform_ = new NodePlatform(thread_pool_size, controller);
    v8::V8::InitializePlatform(platform_);
  }
...
}
这个是 node 对 v8::Platform 的实现，v8::Platform 这个类在 deps/v8/include/v8-platform.h 中有定义
，大概看一下定义了一系列线程相关的虚拟方法
/**
 * V8 Platform abstraction layer.
 *
 * The embedder has to provide an implementation of this interface before
 * initializing the rest of V8.
 */
class Platform {
 public:
  virtual ~Platform() = default;

  /**
   * Gets the number of worker threads used by
   * Call(BlockingTask)OnWorkerThread(). This can be used to estimate the number
   * of tasks a work package should be split into. A return value of 0 means
   * that there are no worker threads available. Note that a value of 0 won't
   * prohibit V8 from posting tasks using |CallOnWorkerThread|.
   */
  virtual int NumberOfWorkerThreads() = 0;

  /**
   * Returns a TaskRunner which can be used to post a task on the foreground.
   * This function should only be called from a foreground thread.
   */
  virtual std::shared_ptr<v8::TaskRunner> GetForegroundTaskRunner(
      Isolate* isolate) = 0;

  /**
   * Schedules a task to be invoked on a worker thread.
   */
  virtual void CallOnWorkerThread(std::unique_ptr<Task> task) = 0;

  /**
   * Schedules a task that blocks the main thread to be invoked with
   * high-priority on a worker thread.
   */
  virtual void CallBlockingTaskOnWorkerThread(std::unique_ptr<Task> task) {
    // Embedders may optionally override this to process these tasks in a high
    // priority pool.
    CallOnWorkerThread(std::move(task));
  }

  /**
   * Schedules a task to be invoked with low-priority on a worker thread.
   */
  virtual void CallLowPriorityTaskOnWorkerThread(std::unique_ptr<Task> task) {
    // Embedders may optionally override this to process these tasks in a low
    // priority pool.
    CallOnWorkerThread(std::move(task));
  }

  /**
   * Schedules a task to be invoked on a worker thread after |delay_in_seconds|
   * expires.
   */
  virtual void CallDelayedOnWorkerThread(std::unique_ptr<Task> task,
                                         double delay_in_seconds) = 0;
  ...
}

V8::Initialize
很显然，这个方法是初始化 V8，可以看到代码在 deps/v8/src/init/v8.cc 中
bool V8::Initialize() {
  InitializeOncePerProcess();
  return true;
}

void V8::InitializeOncePerProcess() {
  base::CallOnce(&init_once, &InitializeOncePerProcessImpl);
}
CallOnce 确保只运行一次，看一下 InitializeOncePerProcessImpl 的实现
void V8::InitializeOncePerProcessImpl() {
  ...
  base::OS::Initialize(FLAG_hard_abort, FLAG_gc_fake_mmap);
  ...
  Isolate::InitializeOncePerProcess();
#if defined(USE_SIMULATOR)
  Simulator::InitializeOncePerProcess();
#endif
  CpuFeatures::Probe(false);
  ElementsAccessor::InitializeOncePerProcess();
  Bootstrapper::InitializeOncePerProcess();
  CallDescriptors::InitializeOncePerProcess();
  wasm::WasmEngine::InitializeOncePerProcess();
}
这里对 V8 的一些基础概念进行初始化操作，例如 Isolate，主要还是初始化一些线程的参数
base::Thread::LocalStorageKey Isolate::isolate_key_;
base::Thread::LocalStorageKey Isolate::per_isolate_thread_data_key_;

void Isolate::InitializeOncePerProcess() {
  isolate_key_ = base::Thread::CreateThreadLocalKey();
  ...
  per_isolate_thread_data_key_ = base::Thread::CreateThreadLocalKey();
}
V8 后续的内容就不多介绍了，有兴趣可以继续研究~

NodeMainInstance
下面看一下 NodeMainInstance，这个的定义在 src/node_main_instance.h 中有定义，主要列一下构造函数、Run 方法、和 private 成员变量
class NodeMainInstance {
 public:
  // Create a main instance that owns the isolate
  NodeMainInstance(
      v8::Isolate::CreateParams* params,
      uv_loop_t* event_loop,
      MultiIsolatePlatform* platform,
      const std::vector<std::string>& args,
      const std::vector<std::string>& exec_args,
      const std::vector<size_t>* per_isolate_data_indexes = nullptr);
  ~NodeMainInstance();

  // Start running the Node.js instances, return the exit code when finished.
  int Run();

 private:
  std::vector<std::string> args_;
  std::vector<std::string> exec_args_;
  std::unique_ptr<ArrayBufferAllocator> array_buffer_allocator_;
  v8::Isolate* isolate_;
  MultiIsolatePlatform* platform_;
  std::unique_ptr<IsolateData> isolate_data_;
  bool owns_isolate_ = false;
  bool deserialize_mode_ = false;
};

下面看一下 cc 文件中的实现
Constructor
构造函数首先将参数赋值给成员变量，然后 alloc 一个 v8 的 Isolate，调用 platform->RegisterIsolate 和 isolate 自身的 Initialize 方法，初始化 IsolateData
NodeMainInstance::NodeMainInstance(
    Isolate::CreateParams* params,
    uv_loop_t* event_loop,
    MultiIsolatePlatform* platform,
    const std::vector<std::string>& args,
    const std::vector<std::string>& exec_args,
    const std::vector<size_t>* per_isolate_data_indexes)
    : args_(args),
      exec_args_(exec_args),
      array_buffer_allocator_(ArrayBufferAllocator::Create()),
      isolate_(nullptr),
       platform_(platform),
      isolate_data_(nullptr),
      owns_isolate_(true) {
  params->array_buffer_allocator = array_buffer_allocator_.get();
  isolate_ = Isolate::Allocate();
  CHECK_NOT_NULL(isolate_);
  // Register the isolate on the platform before the isolate gets initialized,
  // so that the isolate can access the platform during initialization.
  platform->RegisterIsolate(isolate_, event_loop);
  SetIsolateCreateParamsForNode(params);
  Isolate::Initialize(isolate_, *params);

  ...

  isolate_data_.reset(new IsolateData(isolate_,
                                      event_loop,
                                      platform,
                                      array_buffer_allocator_.get(),
                                      per_isolate_data_indexes));
  ...
  SetIsolateUpForNode(isolate_, IsolateSettingCategories::kMisc);
}
SetIsolateUpForNode
注意一下最后的 SetIsolateUpForNode，将 V8 中的事件与 node 主进程绑定，可以看到一些比较容易理解的例如 SetAbortOnUncaughtExceptionCallback、SetFatalErrorHandler、SetPromiseRejectCallback
void SetIsolateUpForNode(v8::Isolate* isolate, IsolateSettingCategories cat) {
  switch (cat) {
    case IsolateSettingCategories::kErrorHandlers:
      isolate->AddMessageListenerWithErrorLevel(
          errors::PerIsolateMessageListener,
          Isolate::MessageErrorLevel::kMessageError |
              Isolate::MessageErrorLevel::kMessageWarning);
      isolate->SetAbortOnUncaughtExceptionCallback(
          ShouldAbortOnUncaughtException);
      isolate->SetFatalErrorHandler(OnFatalError);
      isolate->SetPrepareStackTraceCallback(PrepareStackTraceCallback);
      break;
    case IsolateSettingCategories::kMisc:
      isolate->SetMicrotasksPolicy(MicrotasksPolicy::kExplicit);
      isolate->SetAllowWasmCodeGenerationCallback(
          AllowWasmCodeGenerationCallback);
      isolate->SetPromiseRejectCallback(task_queue::PromiseRejectCallback);
      v8::CpuProfiler::UseDetailedSourcePositionsForProfiling(isolate);
      break;
    default:
      UNREACHABLE();
      break;
  }
}
Run
首先是 v8 惯例的 isolate 和 scope
int NodeMainInstance::Run() {
  Locker locker(isolate_);
  Isolate::Scope isolate_scope(isolate_);
  HandleScope handle_scope(isolate_);

  int exit_code = 0;
  std::unique_ptr<Environment> env = CreateMainEnvironment(&exit_code);

  if (exit_code == 0) {
    {
      AsyncCallbackScope callback_scope(env.get());
      env->async_hooks()->push_async_ids(1, 0);
      LoadEnvironment(env.get());
      env->async_hooks()->pop_async_id(1);
    }
    SealHandleScope seal(isolate_);
    bool more;
    env->performance_state()->Mark(
          node::performance::NODE_PERFORMANCE_MILESTONE_LOOP_START);
    do {
      uv_run(env->event_loop(), UV_RUN_DEFAULT);
      ...
      more = uv_loop_alive(env->event_loop());
      if (more && !env->is_stopping()) continue;
      ...
      more = uv_loop_alive(env->event_loop());
    } while (more == true && !env->is_stopping());
    env->performance_state()->Mark(
          node::performance::NODE_PERFORMANCE_MILESTONE_LOOP_EXIT);    
  }
  return exit_code;
}

之后一共执行了代码：
- CreateMainEnvironment
- 处理 async_hooks
- LoadEnvironment
- 执行 libuv 循环以及记录 performance

CreateMainEnvironment
看一下 CreateMainEnvironment 做了什么
std::unique_ptr<Environment> NodeMainInstance::CreateMainEnvironment(
    int* exit_code) {
  *exit_code = 0;  // Reset the exit code to 0

  HandleScope handle_scope(isolate_);

  Local<Context> context;
  if (deserialize_mode_) {
    context =
        Context::FromSnapshot(isolate_, kNodeContextIndex).ToLocalChecked();
    InitializeContextRuntime(context);
    SetIsolateUpForNode(isolate_, IsolateSettingCategories::kErrorHandlers);
  } else {
    context = NewContext(isolate_);
  }

  CHECK(!context.IsEmpty());
  Context::Scope context_scope(context);

  std::unique_ptr<Environment> env = std::make_unique<Environment>(
      isolate_data_.get(),
      context,
      args_,
      exec_args_,
      static_cast<Environment::Flags>(Environment::kIsMainThread |
                                      Environment::kOwnsProcessState |
                                      Environment::kOwnsInspector));
  env->InitializeLibuv(per_process::v8_is_profiling);
  env->InitializeDiagnostics();

  // TODO(joyeecheung): when we snapshot the bootstrapped context,
  // the inspector and diagnostics setup should after after deserialization.
#if HAVE_INSPECTOR && NODE_USE_V8_PLATFORM
  *exit_code = env->InitializeInspector(nullptr);
#endif
  if (*exit_code != 0) {
    return env;
  }
  if (env->RunBootstrapping().IsEmpty()) {
    *exit_code = 1;
  }
  return env;
}
- 创建和初始化 v8 的 Context
- 创建 Environment
- 初始化 Environment 的 libuv、diagnostics、inspector 等
- 执行 Environment 的 RunBootstrapping
Environment 初始化的部分不多介绍，重点看 RunBootstrapping 的实现，可以找到在 src/node.cc 中，主要执行了 BootstrapInternalLoaders 和 BootstrapNode 这两个方法
MaybeLocal<Value> Environment::RunBootstrapping() {
  EscapableHandleScope scope(isolate_);

  CHECK(!has_run_bootstrapping_code());

  if (BootstrapInternalLoaders().IsEmpty()) {
    return MaybeLocal<Value>();
  }

  Local<Value> result;
  if (!BootstrapNode().ToLocal(&result)) {
    return MaybeLocal<Value>();
  }
  ...
  return scope.Escape(result);
}

BootstrapInternalLoaders
先看 BootstrapInternalLoaders，可以看到从这往后的 namespace 基本上就是 V8 的
MaybeLocal<Value> Environment::BootstrapInternalLoaders() {
  EscapableHandleScope scope(isolate_);

  // Create binding loaders
  std::vector<Local<String>> loaders_params = {
      process_string(),
      FIXED_ONE_BYTE_STRING(isolate_, "getLinkedBinding"),
      FIXED_ONE_BYTE_STRING(isolate_, "getInternalBinding"),
      primordials_string()};
  std::vector<Local<Value>> loaders_args = {
      process_object(),
      NewFunctionTemplate(binding::GetLinkedBinding)
          ->GetFunction(context())
          .ToLocalChecked(),
      NewFunctionTemplate(binding::GetInternalBinding)
          ->GetFunction(context())
          .ToLocalChecked(),
      primordials()};

  // Bootstrap internal loaders
  Local<Value> loader_exports;
  if (!ExecuteBootstrapper(
           this, "internal/bootstrap/loaders", &loaders_params, &loaders_args)
           .ToLocal(&loader_exports)) {
    return MaybeLocal<Value>();
  }
  CHECK(loader_exports->IsObject());
  Local<Object> loader_exports_obj = loader_exports.As<Object>();
  Local<Value> internal_binding_loader =
      loader_exports_obj->Get(context(), internal_binding_string())
          .ToLocalChecked();
  CHECK(internal_binding_loader->IsFunction());
  set_internal_binding_loader(internal_binding_loader.As<Function>());
  Local<Value> require =
      loader_exports_obj->Get(context(), require_string()).ToLocalChecked();
  CHECK(require->IsFunction());
  set_native_module_require(require.As<Function>());

  return scope.Escape(loader_exports);
}
可以看到主要的代码在 ExecuteBootstrapper "internal/bootstrap/loaders"
注意一下这里注入了两个变量：
- getLinkedBinding
- getInternalBinding
作用后面再介绍

ExecuteBootstrapper
追踪一下 ExecuteBootstrapper
MaybeLocal<Value> ExecuteBootstrapper(Environment* env,
                                      const char* id,
                                      std::vector<Local<String>>* parameters,
                                      std::vector<Local<Value>>* arguments) {
  EscapableHandleScope scope(env->isolate());
  MaybeLocal<Function> maybe_fn =
      NativeModuleEnv::LookupAndCompile(env->context(), id, parameters, env);
  ...
  Local<Function> fn = maybe_fn.ToLocalChecked();
  MaybeLocal<Value> result = fn->Call(env->context(),
                                      Undefined(env->isolate()),
                                      arguments->size(),
                                      arguments->data());
  ...
  return scope.EscapeMaybe(result);
}
再看一下 LookupAndCompile，依次在 src/node_native_module_env.cc 和 src/node_native_module.cc 中
MaybeLocal<Function> NativeModuleLoader::LookupAndCompile(
    Local<Context> context,
    const char* id,
    std::vector<Local<String>>* parameters,
    NativeModuleLoader::Result* result) {
  Isolate* isolate = context->GetIsolate();
  EscapableHandleScope scope(isolate);

  const auto source_it = source_.find(id);
  CHECK_NE(source_it, source_.end());
  Local<String> source = source_it->second.ToStringChecked(isolate);

  ...

  ScriptCompiler::Source script_source(source, origin, cached_data);

  MaybeLocal<Function> maybe_fun =
      ScriptCompiler::CompileFunctionInContext(context,
                                               &script_source,
                                               parameters->size(),
                                               parameters->data(),
                                               0,
                                               nullptr,
                                               options);
  ...

  Local<Function> fun = maybe_fun.ToLocalChecked();

  ...

  return scope.Escape(fun);
}
可以看出来这里是从 source_ 中去找到对应的 js 字符串，使用 v8 的 ScriptCompiler 完成 js 代码解释
在头文件中看到 source_ 的定义如下：
using NativeModuleRecordMap = std::map<std::string, UnionBytes>;
...

class NativeModuleLoader {
  ...
  // Generated by tools/js2c.py as node_javascript.cc
  void LoadJavaScriptSource();  // Loads data into source_
  ...
  NativeModuleRecordMap source_;
  ...
}

注意这里代码的注释，再回到之前 gyp 编译的时候 js 文件是 js2c.py 处理的

js2c
详细看一下 js2c.py 的处理过程：
- main 函数读取命令行参数，结合之前 node.gyp 里的配置可以看到，input 就是所有的 js 文件，output 是 node_javascript.cc
def main():
  parser = argparse.ArgumentParser(
    description='Convert code files into `uint16_t[]`s',
    fromfile_prefix_chars='@'
  )
  parser.add_argument('--target', help='output file')
  parser.add_argument('--verbose', action='store_true', help='output file')
  parser.add_argument('sources', nargs='*', help='input files')
  options = parser.parse_args()
  ...
  source_files = functools.reduce(SourceFileByExt, options.sources, {})
  ...
  JS2C(source_files, options.target)
- JS2C 函数读取所有文件，填充模板字符串，写入文件

def JS2C(source_files, target):
  # Process input from all *macro.py files
  consts, macros = ReadMacros(source_files['.py'])

  # Build source code lines
  definitions = []
  initializers = []

  for filename in source_files['.js']:
    AddModule(filename, consts, macros, definitions, initializers)
  ...
  # Emit result
  definitions = ''.join(definitions)
  initializers = '\n  '.join(initializers)
  out = TEMPLATE.format(definitions, initializers, config_size)
  write_if_chaged(out, target)
- C++ 的模板字符串
TEMPLATE = """
#include "env-inl.h"
#include "node_native_module.h"
#include "node_internals.h"

namespace node {{

namespace native_module {{

{0}

void NativeModuleLoader::LoadJavaScriptSource() {{
  {1}
}}

UnionBytes NativeModuleLoader::GetConfig() {{
  return UnionBytes(config_raw, {2});  // config.gypi
}}

}}  // namespace native_module

}}  // namespace node
"""
- 看一下编译后的 out/Release/obj/gen/node_javascript.cc，js 文件被转成了 ascii 码，最后在 LoadJavaScriptSource 加载到 source_ 变量中
#include "env-inl.h"
#include "node_native_module.h"
#include "node_internals.h"

namespace node {

namespace native_module {


static const uint8_t internal_bootstrap_environment_raw[] = {
 39,117,115,101, 32,115,116,114,105, 99,116, 39, 59, 10, 10, 47, 47, 32, 84,104,105,115, 32,114,117,110,115, 32,110,101,
 99,101,115,115, 97,114,121, 32,112,114,101,112, 97,114, 97,116,105,111,110,115, 32,116,111, 32,112,114,101,112, 97,114,
101, 32, 97, 32, 99,111,109,112,108,101,116,101, 32, 78,111,100,101, 46,106,115, 32, 99,111,110,116,101,120,116, 10, 47,
 47, 32,116,104, 97,116, 32,100,101,112,101,110,100,115, 32,111,110, 32,114,117,110, 32,116,105,109,101, 32,115,116, 97,
116,101,115, 46, 10, 47, 47, 32, 73,116, 32,105,115, 32, 99,117,114,114,101,110,116,108,121, 32,111,110,108,121, 32,105,
110,116,101,110,100,101,100, 32,102,111,114, 32,112,114,101,112, 97,114,105,110,103, 32, 99,111,110,116,101,120,116,115,
 32,102,111,114, 32,101,109, 98,101,100,100,101,114,115, 46, 10, 10, 47, 42, 32,103,108,111, 98, 97,108, 32,109, 97,114,
107, 66,111,111,116,115,116,114, 97,112, 67,111,109,112,108,101,116,101, 32, 42, 47, 10, 99,111,110,115,116, 32,123, 10,
 32, 32,112,114,101,112, 97,114,101, 77, 97,105,110, 84,104,114,101, 97,100, 69,120,101, 99,117,116,105,111,110, 10,125,
 32, 61, 32,114,101,113,117,105,114,101, 40, 39,105,110,116,101,114,110, 97,108, 47, 98,111,111,116,115,116,114, 97,112,
 47,112,114,101, 95,101,120,101, 99,117,116,105,111,110, 39, 41, 59, 10, 10,112,114,101,112, 97,114,101, 77, 97,105,110,
 84,104,114,101, 97,100, 69,120,101, 99,117,116,105,111,110, 40, 41, 59, 10,109, 97,114,107, 66,111,111,116,115,116,114,
 97,112, 67,111,109,112,108,101,116,101, 40, 41, 59, 10
};

...


void NativeModuleLoader::LoadJavaScriptSource() {
  source_.emplace("internal/bootstrap/environment", UnionBytes{internal_bootstrap_environment_raw, 374});
  source_.emplace("internal/bootstrap/loaders", UnionBytes{internal_bootstrap_loaders_raw, 9811});
  source_.emplace("internal/bootstrap/node", UnionBytes{internal_bootstrap_node_raw, 15835});
  source_.emplace("internal/bootstrap/pre_execution", UnionBytes{internal_bootstrap_pre_execution_raw, 14393});
  ...
}

UnionBytes NativeModuleLoader::GetConfig() {
  return UnionBytes(config_raw, 3028);  // config.gypi
}

}  // namespace native_module

}  // namespace node
至此就很清晰了，之前 BootstrapInternalLoaders 其实是运行了 internal/bootstrap/loaders.js 文件
简单看一下 loaders.js 文件可以发现初始化了所有的 node 内建模块
// Set up internalBinding() in the closure.
let internalBinding;
{
  const bindingObj = Object.create(null);
  // eslint-disable-next-line no-global-assign
  internalBinding = function internalBinding(module) {
    let mod = bindingObj[module];
    if (typeof mod !== 'object') {
      mod = bindingObj[module] = getInternalBinding(module);
      moduleLoadList.push(`Internal Binding ${module}`);
    }
    return mod;
  };
}

function NativeModule(id) {
  this.filename = `${id}.js`;
  this.id = id;
  this.exports = {};
  this.reflect = undefined;
  this.exportKeys = undefined;
  this.loaded = false;
  this.loading = false;
  this.canBeRequiredByUsers = !id.startsWith('internal/');
}

NativeModule.map = new Map();
for (let i = 0; i < moduleIds.length; ++i) {
  const id = moduleIds[i];
  const mod = new NativeModule(id);
  NativeModule.map.set(id, mod);
}

getInternalBinding
注意我们之前提到了 BootstrapInternalLoaders 注入了 getInternalBinding 和 getLinkedBinding，这里根据 getInternalBinding 封装了一个 internalBinding，这个方法在 js 中经常用到，用于在 js 中加载 c++ 模块

BootstrapNode
再看 BootstrapNode 就很容易理解了，跟 BootstrapInternalLoaders 类似地运行了 internal/bootstrap/node.js 文件
MaybeLocal<Value> Environment::BootstrapNode() {
  EscapableHandleScope scope(isolate_);
  ...
  MaybeLocal<Value> result = ExecuteBootstrapper(
      this, "internal/bootstrap/node", &node_params, &node_args);
  ...
  return scope.EscapeMaybe(result);
}

LoadEnvironment
回到最开始的 Run 方法，async_hooks 不多介绍，看一下 LoadEnvironment

简单追踪以下就可以看到 LoadEnvironment 执行了 internal/main 下的 js 文件，这里跟前面的差不多也不重复说明
void LoadEnvironment(Environment* env) {
  CHECK(env->is_main_thread());
  USE(StartMainThreadExecution(env));
}

MaybeLocal<Value> StartMainThreadExecution(Environment* env) {
  if (NativeModuleEnv::Exists("_third_party_main")) {
    return StartExecution(env, "internal/main/run_third_party_main");
  }

  std::string first_argv;
  if (env->argv().size() > 1) {
    first_argv = env->argv()[1];
  }

  if (first_argv == "inspect" || first_argv == "debug") {
    return StartExecution(env, "internal/main/inspect");
  }

  ...

  if (env->options()->force_repl || uv_guess_handle(STDIN_FILENO) == UV_TTY) {
    return StartExecution(env, "internal/main/repl");
  }

  return StartExecution(env, "internal/main/eval_stdin");
}


EventLoop
Run 之后的逻辑就是执行 libuv 的事件循环了，可以看到只要事件循环保持 alive，while 循环就不会结束，程序也不会退出。
这一部分基本是调用 libuv 的 API，不多做介绍，感兴趣可以继续关注 deps/libuv 里的代码
do {
  uv_run(env->event_loop(), UV_RUN_DEFAULT);
  ...
  more = uv_loop_alive(env->event_loop());
  if (more && !env->is_stopping()) continue;
  ...
  more = uv_loop_alive(env->event_loop());
} while (more == true && !env->is_stopping());

小结
通过前面的介绍，相信大家对代码结构有一个初步的了解了，总结一下：
- node 主要是 C++ 代码实现的，核心代码在 src 目录下
- v8、libuv 等第三方依赖库在 deps 目录下，由 node 的 C++ 代码负责调用
- js 是 node 运行过程中由 v8 完成动态解释运行的，node 核心的 js 代码在 lib 目录下
- python 主要是工具类代码，负责完成编译等能力
