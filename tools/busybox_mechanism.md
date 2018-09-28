# Busybox mechanism
1. busybox main entry is /libbb/appletlib.c/main
2. Two usages for busybox
	1. busybox ls
	```
	main
	-->applet_name = bb_basename(applet_name)   // applet_name = busybox
	--> run_applet_and_exit
	-->busybox_main(argv)
	-->applet_name = bb_get_last_path_component_nostrip(argv[0]);    // applet_name = ls
	-->run_applet_and_exit
	-->int applet = find_applet_by_name(name);
	-->run_applet_no_and_exit
	-->xfunc_error_retval = applet_main[applet_no](argc, argv);
	-->ls_main
	```
	2. ls (Software link)
	```
	main
	-->applet_name = bb_basename(applet_name);
	-->run_applet_and_exit(applet_name, argv);
	-->find_applet_by_name(name);
	-->run_applet_no_and_exit(applet, argv)
	-->xfunc_error_retval = applet_main[applet_no](argc, argv);
	-->ls_main
	```