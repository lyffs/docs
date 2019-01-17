# GRACE #

-> github地址：
	https://github.com/facebookgo/grace


*1.1分析*

-> 继承（inherit）
	// 根据指定的文件描述符和名称 返回文件
	file := os.NewFile(uintptr(i), "listener")

	// 返回与文件关联的network listener的副本
	l, err := net.FileListener(file) 

	// 启动新进程
	pallFiles := append([]*os.File{os.Stdin, os.Stdout, os.Stderr}, files...)
	process, err := os.StartProcess(argv0, os.Args, &os.ProcAttr{
		Dir:   originalWD,
		Env:   env,
		Files: allFiles,
	})

	->StartProcess 启动中的新进程，文件描述符一般是从3开始（after stdin, stdout, stderr），以参数Files 对应

