<!--
author: zhuoliang
head: http://pingodata.qiniudn.com/jockchou-avatar.jpg
date: 2016-08-01
title: Issues on C&C++
tags: C&C++, Coding
category: Coding
status: publish
-->


## Wide Code Set To Muiti Bytes Set ##

    std::wstring wcs_arg_str((wchar_t*)start);
	std::string mbs_arg_str;
	mbs_arg_str.assign(wcs_arg_str.begin(), wcs_arg_str.end()); 


## Simple Log For Debug&Release ##

	#ifdef DEBUG
	#define LOG(stream, format, args...) fprintf(stream, format, ##args)
	#else
	#define LOG(stream, format, args...) 
	#endif /* DEBUG */

## Actions on Loading&Unloading Dynamic Library ##

**Linux**

    void onload(void) __attribute__((constructor))
	{
		/* Things to do on load */
	}

	void unload(void) __attribute__((destructor))
	{
		/* Things to do unload */
	}
 
**Windows**






	






						