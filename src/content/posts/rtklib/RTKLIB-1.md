---
title: RTKLIB精密定位配置
published: 2021-11-21
description: '本节将给出RTKLIB的差分定位与精密单点定位使用示例，通过本节可以了解到RTKLIB如何处理数据，并对上一节留下的一些问题进行补充，比如某个类的变量在真实数据处理当中的作用'
image: ''
tags: [GNSS, RTKLIB]
category: 'GNSS'
draft: false 
lang: ''
---

<font color=#33CCCC>**这里有个写单点定位很好的博客，也补充了很多函数的解释：**</font>
- [RTKLIB单点定位处理流程之一（postpos/后处理）](https://blog.csdn.net/wuwuku123/article/details/106068946/)

***
# 一、精密单点定位

<font color=#9900CC size=3>**所选用的数据为[武汉大学IGS数据中心](http://www.igs.gnsswhu.cn/index.php/home/data_product/igs.html)上下载的2020年9月25日长春站（chan）的观测数据，同时下载了当天的广播星历以及精密星历。（chan2690.20o、brdc2690.20n、igs21245.sp3）**</font>
<br>

***
<font color=teal>**这里直接给出代码，在注释中进行解释,注意设置的计算历元时段，如果设置时间不在所选用的数据的历元内不会输出结果。在RTKLIB当中，导航电文即广播星历与观测值文件在任何定位模式下都是必须存在的，其他文件是可选的。**</font>
***
```c
int main() {
	int i, n, ret;
	double tint = 0.0;       /* 求解时间间隔(0:默认) */
	gtime_t ts = { 0 }, te = { 0 }; /* 历元时段始末控制变量 */
	char *infile[MAXFILE], outfile[MAXSTRPATH] = { '\0' };
	char resultpath[MAXSTRPATH] = "H:\\20211108\\result"; /* 结果输出路径 */
	char sep = (char)FILEPATHSEP;
	prcopt_t prcopt = prcopt_default; /* 默认处理选项设置 */
	solopt_t solopt = solopt_default; /* 默认求解格式设置 */
	filopt_t filopt = { /* 参数文件路径设置 */
		"", /* 卫星天线参数文件 */
		"", /* 接收机天线参数文件 */
		"", /* 测站位置文件 */
		"", /* 扩展大地水准面数据文件 */
		"", /* 电离层数据文件 */
		"", /* DCB数据文件 */
		"", /* 地球自转参数文件 */
		"", /* 海洋潮汐负荷文件 */
	};
	char infile_[MAXFILE][MAXSTRPATH] = {
		"H:\\20211108\\chan2690.20o",
		"H:\\20211108\\brdc2690.20n",
		"H:\\20211108\\igs21245.sp3",
		"",
		"",
		"",
		"",
		""
	};
	long t1, t2;
	double eps[]={2020,9,25,0,0,0},epe[]={2020,9,25,23,0,0}; /* 设置计算的历元时段 */
	ts=epoch2time(eps);te=epoch2time(epe);

	for (i = 0, n = 0; i < MAXFILE; i++)
		if (strcmp(infile_[i], "")) infile[n++] = &infile_[i][0];

	sprintf(outfile, "%s%c", resultpath, sep);//设置输出路径

	/* 自定义求解格式 --------------------------------------------------------*/
	solopt.posf = SOLF_XYZ;   /* 选择输出的坐标格式，经纬度或是XYZ坐标等 */
	solopt.times  =TIMES_UTC; /* 控制输出解的时间系统类型 */
	solopt.degf   =0;         /* 输出经纬度格式(0:°, 1:°′″) */
	solopt.outhead=1;         /* 是否输出头文件(0:否,1:是) */
	solopt.outopt =1;         /* 是否输出prcopt变量(0:否,1:是) */
	solopt.height =1;         /* 高程(0:椭球高,1:大地高) */

	/* 自定义处理选项设置 ----------------------------------------------------*/
	prcopt.mode = PMODE_PPP_KINEMA; /* PPP动态处理 */
	prcopt.modear = 4;     /* 求解模糊度类型 */
	prcopt.sateph = EPHOPT_PREC;      /* 使用精密星历 */
	prcopt.ionoopt = IONOOPT_IFLC;     /* 使用双频消电离层组合模型 */
	prcopt.tropopt = TROPOPT_EST;      /* 使用对流层天顶延迟估计模型 */
	prcopt.tidecorr = 0; /* 地球潮汐改正选项(0:关闭,1:固体潮,2:固体潮+?+极移) */
	prcopt.posopt[0] = 0; /* 卫星天线模型 */
	prcopt.posopt[1] = 0; /* 接收机天线模型 */
	prcopt.posopt[2] = 0; /* 相位缠绕改正 */
	prcopt.posopt[3] = 0; /* 排除掩星 */
	prcopt.posopt[4] = 0; /* 求解接收机坐标出错后的检查选项 */
	prcopt.navsys = SYS_GPS; /* 处理的导航系统 */
	sprintf(outfile, "%s%cChan200925.pos", resultpath, sep); /* 输出结果名称 */
	prcopt.nf       =2;       /* 参与计算的载波频率个数 */
	prcopt.elmin    =10.0*D2R;/* 卫星截止高度角 */
	prcopt.soltype  =0;       /* 求解类型(0:向前滤波,1:向后滤波,2:混合滤波) */

	t1 = clock();
	ret = postpos(ts, te, tint, 0.0, &prcopt, &solopt, &filopt, infile, n, outfile, "", "");
	t2 = clock();

	if (!ret) fprintf(stderr, "%40s\r", "");

	printf("\n * The total time for running the program: %6.3f seconds\n%c", (double)(t2 - t1) / CLOCKS_PER_SEC, '\0');
	printf("Press any key to exit!\n");
	getchar();
	return ret;
}
```
***

# 二、差分定位

<font color=#9900CC size=3>**所选用的数据为[武汉大学IGS数据中心](http://www.igs.gnsswhu.cn/index.php/home/data_product/igs.html)上下载的2020年9月25日长春站（chan）与北京佛山站（bjfs）的观测数据，同时下载了当天的广播星历以及精密星历。（chan2690.20o、bjfs2690.20o、brdc2690.20n、igs21245.sp3）**</font>
***
<font color=teal>**在差分定位中，需要至少两个站的观测值，在RTKLIB中只有第一个观测值文件会被当做移动站处理，这里笔者将chan作为移动站，bjfs作为基准站**</font>
***
```c
int main() {
	int i, n, ret;
	double tint = 0.0;       /* 求解时间间隔(0:默认) */
	gtime_t ts = { 0 }, te = { 0 }; /* 历元时段始末控制变量 */
	char *infile[MAXFILE], outfile[MAXSTRPATH] = { '\0' };
	char resultpath[MAXSTRPATH] = "H:\\20211108\\result"; /* 结果输出路径 */
	char sep = (char)FILEPATHSEP;
	prcopt_t prcopt = prcopt_default; /* 默认处理选项设置 */
	solopt_t solopt = solopt_default; /* 默认求解格式设置 */
	filopt_t filopt = { /* 参数文件路径设置 */
		"", /* 卫星天线参数文件 */
		"", /* 接收机天线参数文件 */
		"", /* 测站位置文件 */
		"", /* 扩展大地水准面数据文件 */
		"", /* 电离层数据文件 */
		"", /* DCB数据文件 */
		"", /* 地球自转参数文件 */
		"", /* 海洋潮汐负荷文件 */
	};
	char infile_[MAXFILE][MAXSTRPATH] = { /* 前面观测值为移动站,后面观测值为基准站 */
		"H:\\20211108\\chan2690.20o",
		"H:\\20211108\\brdc2690.20n",
		"H:\\20211108\\igs21245.sp3",
		"H:\\20211108\\bjfs2690.20o",
		"",
		"",
		"",
		""
	};
	long t1, t2;
	double eps[]={2020,9,25,0,0,0},epe[]={2020,9,25,23,0,0}; /* 设置计算的历元时段 */
	ts=epoch2time(eps);te=epoch2time(epe);

	for (i = 0, n = 0; i < MAXFILE; i++)
		if (strcmp(infile_[i], "")) infile[n++] = &infile_[i][0];

	sprintf(outfile, "%s%c", resultpath, sep);

	/* 自定义求解格式 --------------------------------------------------------*/
	solopt.timef = 1;       /* 时间格式(0:sssss.s, 1:yyyy/mm/dd hh:mm:ss.s) */
	solopt.outhead = 1;       /* 是否输出头文件(0:否,1:是) */
	solopt.posf = SOLF_XYZ;  /* 输出的坐标格式 */
	//solopt.sstat =1;         /* 输出状态文件 */
	solopt.times =TIMES_UTC; /* 控制输出解的时间系统类型 */
	//solopt.degf  =0;         /* 输出经纬度格式(0:°, 1:°′″) */
	solopt.outopt=1;         /* 是否输出prcopt变量(0:否,1:是) */
	solopt.height=1;         /* 高程(0:椭球高,1:大地高) */
	//solopt.sstat =1;         /* 输出求解状态 */

	/* 自定义处理选项设置 ----------------------------------------------------*/
	prcopt.mode = PMODE_DGPS; /* 差分GPS处理 */
	prcopt.modear = ARMODE_FIXHOLD;  /* 求解模糊度类型 */
	prcopt.sateph = EPHOPT_BRDC; /* 使用广播星历 */
	prcopt.ionoopt = IONOOPT_BRDC;  /* 使用广播电离层模型 */
	prcopt.tropopt = TROPOPT_SAAS;  /* 使用萨斯坦莫宁模型 */
	prcopt.refpos =3;       /* 相对模式中基站位置获得方式 */
							/* (0:pos in prcopt,  1:average of single pos, */
							/*  2:read from file, 3:rinex header, 4:rtcm pos) */
	//freqindex     =0;       /* 单频计算时设置所用计算的波段 */
	prcopt.nf     =2;       /* 参与计算的载波频率个数 L1+L2*/
	//prcopt.glomodear =2;
	//prcopt.thresar[0]=2;
	//prcopt.posopt[4]=1;     /* 求解接收机坐标出错后的检查选项 */
	//prcopt.intpref=1;
	prcopt.elmin  =10.0*D2R;/* 卫星截止高度角 */
	prcopt.soltype=0;       /* 求解类型(0:向前滤波,1:向后滤波,2:混合滤波) */


	prcopt.navsys = SYS_GPS; /* 处理的导航系统 */
	sprintf(outfile, "%s%cchan200925DGPS.pos", resultpath, sep); /* 输出结果名称 */

	//prcopt.navsys =SYS_GPS|SYS_CMP; /* 处理的导航系统 */
	//sprintf(outfile,"%s%cresult_MIX.txt",resultpath,sep);

	t1 = clock();
	ret = postpos(ts, te, tint, 0.0, &prcopt, &solopt, &filopt, infile, n, outfile, "", "");
	t2 = clock();

	if (!ret) fprintf(stderr, "%40s\r", "");

	printf("\n * The total time for running the program: %6.3f seconds\n%c", (double)(t2 - t1) / CLOCKS_PER_SEC, '\0');
	printf("Press any key to exit!\n");
	getchar();
	return ret;
}
```

