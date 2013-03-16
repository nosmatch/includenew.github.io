---
layout: post
title: "解决Mac上解压Windows压缩包乱码问题"
date: 2012-02-13 16:53
comments: true
categories: Ruby
---

Windows上默认使用的GBK编码，Mac上默认使用的unicode编码，因此Win上的压缩包再Mac上解压会出现文件名乱码:(
下面是用ruby写的一个解决方法：

[source@github](https://github.com/jiang-bo/codingforfun/blob/master/ruby/utils/unzipFromWinToMac.rb)

	require 'zip/zip'
	require 'iconv'
	
	# To unzip zipfile which zip in GBK to UTF-8.
	#
	# When you zip a file on Windows, it will encode in GBK default.
	# Then you unzip it on Mac OSX which use unicode default, it will be wrong.
	# This code is used to fix this problem:)
	#
	# @Author: jiang-bo
	Zip::ZipInputStream::open(zipFile){
	  |io|
	  while(entry = io.get_next_entry)
	    name=Iconv.iconv("UTF-8","GBK", entry.name)[0]
	
	    puts "Extracting #{name}"
	    if name.end_with?('/')
	      Dir.mkdir(name.to_s)
	    else
	      entry.extract(name.to_s)
	    end
	  end
	}

主要依赖rubyzip和iconv两个包：

	gem install rubyzip
	gem install iconv

使用ruby zip中ZipInputStream打开压缩包，然后使用Iconv.iconv将其中的文件名由'GBK'转码为'UTF-8'。
然后解压压缩包。
