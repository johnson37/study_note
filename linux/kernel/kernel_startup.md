# Kernel Startup

```
Head.S
-->start_kernel
-->rest_init
-->kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND);
-->kernel_init
-->do_basic_setup
-->do_initcalls
-->Loop [do_initcall_level(level);] 
-->run_init_process("/sbin/init")


static void __init do_initcall_level(int level)
{
        extern const struct kernel_param __start___param[], __stop___param[];
        initcall_t *fn;

        strcpy(static_command_line, saved_command_line);

        parse_args(initcall_level_names[level],
                   static_command_line, __start___param,
                   __stop___param - __start___param,
                   level, level,
                   repair_env_string);

        for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
                do_one_initcall(*fn);
} 
static void __init do_initcalls(void)
{
        int level;

        for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
                do_initcall_level(level);
} 
```