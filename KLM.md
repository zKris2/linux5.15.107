# kernal模块的加载/卸载
## KO文件
一个.ko文件就是一个模块文件。  
.ko是Linux2.6内核使用的动态连接文件，系统启动时的加载内核模块。
## 内核模块加载
insmod会通过文件系统将.ko文件读到用户空间的一块内存中，然后执行系统调用sys_init_module(),通过vmalloc分配内存空间，然后把内核模块拷贝过来，进行搬移、重定向、富豪到处等过程
## 内核模块卸载
rmmod会通过sys_delete_module系统调用，根据文件找到要卸载模块的指针，确保要卸载模块没有被其他模块依赖，然后找到模块本身的exit函数实现卸载

## module.h
数据结构，模块有关的数据结构存放在include/linux/module.h中
```
struct module {
    enum module_state state;

    /* Member of list of modules */
    struct list_head list;

    /* Unique handle for this module */
    char name[MODULE_NAME_LEN];

#ifdef CONFIG_STACKTRACE_BUILD_ID
    /* Module build ID */
    unsigned char build_id[BUILD_ID_SIZE_MAX];
#endif

    /* Sysfs stuff. */
    struct module_kobject mkobj;
    struct module_attribute *modinfo_attrs;
    const char *version;
    const char *srcversion;
    struct kobject *holders_dir;

    /* Exported symbols */
    const struct kernel_symbol *syms;
    const s32 *crcs;
    unsigned int num_syms;

#ifdef CONFIG_CFI_CLANG
    cfi_check_fn cfi_check;
#endif

    /* Kernel parameters. */
#ifdef CONFIG_SYSFS
    struct mutex param_lock;
#endif
    struct kernel_param *kp;
    unsigned int num_kp;

    /* GPL-only exported symbols. */
    unsigned int num_gpl_syms;
    const struct kernel_symbol *gpl_syms;
    const s32 *gpl_crcs;
    bool using_gplonly_symbols;

#ifdef CONFIG_MODULE_SIG
    /* Signature was verified. */
    bool sig_ok;
#endif

    bool async_probe_requested;

    /* Exception table */
    unsigned int num_exentries;
    struct exception_table_entry *extable;

    /* Startup function. */
    int (*init)(void);

    /* Core layout: rbtree is accessed frequently, so keep together. */
    struct module_layout core_layout __module_layout_align;
    struct module_layout init_layout;

    /* Arch-specific module values */
    struct mod_arch_specific arch;

    unsigned long taints;   /* same bits as kernel:taint_flags */

#ifdef CONFIG_GENERIC_BUG
    /* Support for BUG */
    unsigned num_bugs;
    struct list_head bug_list;
    struct bug_entry *bug_table;
#endif

#ifdef CONFIG_KALLSYMS
    /* Protected by RCU and/or module_mutex: use rcu_dereference() */
    struct mod_kallsyms __rcu *kallsyms;
    struct mod_kallsyms core_kallsyms;

    /* Section attributes */
    struct module_sect_attrs *sect_attrs;

    /* Notes attributes */
    struct module_notes_attrs *notes_attrs;
#endif

    /* The command line arguments (may be mangled).  People like
       keeping pointers to this stuff */
    char *args;

#ifdef CONFIG_SMP
    /* Per-cpu data. */
    void __percpu *percpu;
    unsigned int percpu_size;
#endif
    void *noinstr_text_start;
    unsigned int noinstr_text_size;

#ifdef CONFIG_TRACEPOINTS
    unsigned int num_tracepoints;
    tracepoint_ptr_t *tracepoints_ptrs;
#endif
#ifdef CONFIG_TREE_SRCU
    unsigned int num_srcu_structs;
    struct srcu_struct **srcu_struct_ptrs;
#endif
#ifdef CONFIG_BPF_EVENTS
    unsigned int num_bpf_raw_events;
    struct bpf_raw_event_map *bpf_raw_events;
#endif
#ifdef CONFIG_DEBUG_INFO_BTF_MODULES
    unsigned int btf_data_size;
    void *btf_data;
#endif
#ifdef CONFIG_JUMP_LABEL
    struct jump_entry *jump_entries;
    unsigned int num_jump_entries;
#endif
#ifdef CONFIG_TRACING
    unsigned int num_trace_bprintk_fmt;
    const char **trace_bprintk_fmt_start;
#endif
#ifdef CONFIG_EVENT_TRACING
    struct trace_event_call **trace_events;
    unsigned int num_trace_events;
    struct trace_eval_map **trace_evals;
    unsigned int num_trace_evals;
#endif
#ifdef CONFIG_FTRACE_MCOUNT_RECORD
    unsigned int num_ftrace_callsites;
    unsigned long *ftrace_callsites;
#endif
#ifdef CONFIG_KPROBES
    void *kprobes_text_start;
    unsigned int kprobes_text_size;
    unsigned long *kprobe_blacklist;
    unsigned int num_kprobe_blacklist;
#endif
#ifdef CONFIG_HAVE_STATIC_CALL_INLINE
    int num_static_call_sites;
    struct static_call_site *static_call_sites;
#endif

#ifdef CONFIG_LIVEPATCH
    bool klp; /* Is this a livepatch module? */
    bool klp_alive;

    /* Elf information */
    struct klp_modinfo *klp_info;
#endif

#ifdef CONFIG_PRINTK_INDEX
    unsigned int printk_index_size;
    struct pi_entry **printk_index_start;
#endif

#ifdef CONFIG_MODULE_UNLOAD
    /* What modules depend on me? */
    struct list_head source_list;
    /* What modules do I depend on? */
    struct list_head target_list;

    /* Destruction function. */
    void (*exit)(void);

    atomic_t refcnt;
#endif

#ifdef CONFIG_CONSTRUCTORS
    /* Constructor functions. */
    ctor_fn_t *ctors;
    unsigned int num_ctors;
#endif

#ifdef CONFIG_FUNCTION_ERROR_INJECTION
    struct error_injection_entry *ei_funcs;
    unsigned int num_ei_funcs;
#endif
} ____cacheline_aligned __randomize_layout;
```

在内核中，每一个内核模块信息都由这样的一个module对象来描述。所有的module对象通过list链接在一起。链表的第一个元素由static LIST_HEAD(modules)建立，见kernel/module.c。  

### 结构体简单说明

state 表示module当前的状态，可使用的宏定义有：
- MODULE_STATE_LIVE  
- MODULE_STATE_COMING  
- MODULE_STATE_GOING  

struct list_head list 指向一个链表。

name 数组保存module对象的名称。

param_attrs 指向module可传递的参数名称，及其属性

const struct kernel_symbol *syms 模块导出的符号。

int (*init)(void); void (*exit)(void);两个函数指针，其中init函数在初始化模块的时候调用；exit是在删除模块的时候调用的。

查看include/linux/list.h里面的LIST_HEAD宏定义，可以发现modules变量是struct list_head类型结构，结构内部的next指针和prev指针，初始化时都指向modules本身。对modules链表的操作，受module_mutex和modlist_lock保护。
```
struct list_head {
    struct list_head *next, *prev;
};
```
操作系统初始化时，static LIST_HEAD(modules)已经建立了一个空链表。之后，每装入一个内核模块，则创建一个module结构，并把它链接到modules链表中。