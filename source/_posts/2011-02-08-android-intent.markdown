---
layout: post
title: "Android中的intent调用代码总结"
date: 2011-02-08 21:39:34 +0800
comments: true
categories: android
---
 
### 显示Web网页
 
  	Uri uri = Uri.parse("http://www.android123.com.cn");
  	Intent it = new Intent(Intent.ACTION_VIEW,uri);
  	startActivity(it);
 
### 显示Google地图
 
  	Uri uri = Uri.parse("geo:38.899533,-77.036476");
  	Intent it = new Intent(Intent.Action_VIEW,uri);
  	startActivity(it);
 
### Maps路径规划
 
  	Uri uri = Uri.parse("http://maps.google.com/maps?f=d&saddr=startLat%20startLng&daddr=endLat%20endLng&hl=en");
  	Intent it = new Intent(Intent.ACTION_VIEW,URI);
  	startActivity(it);
 
### 拨打电话
 
  	Uri uri = Uri.parse("tel:xxxxxx");
  	Intent it = new Intent(Intent.ACTION_DIAL, uri);   
  	startActivity(it);  
 
  	Uri uri = Uri.parse("tel.xxxxxx");
  	Intent it =new Intent(Intent.ACTION_CALL,uri);
 
注意需要权限android.permission.CALL_PHONE

### 发送SMS/MMS
 
  	Intent it = new Intent(Intent.ACTION_VIEW);
  	it.putExtra("sms_body", "android开发网欢迎您");
  	it.setType("vnd.android-dir/mms-sms");
  	startActivity(it);  
 
### 发送短信
 
  	Uri uri = Uri.parse("smsto:10086");
  	Intent it = new Intent(Intent.ACTION_SENDTO, uri);
  	it.putExtra("sms_body", "10086"); //正文为10086
  	startActivity(it);  
 
### 发送彩信
 
  	Uri uri = Uri.parse("content://media/external/images/media/10"); //该Uri根据实际情况修改，external代表外部存储即sdcard
  	Intent it = new Intent(Intent.ACTION_SEND);
  	it.putExtra("sms_body", "android123.com.cn"); 
  	it.putExtra(Intent.EXTRA_STREAM, uri);
  	it.setType("image/png");
  	startActivity(it);
 
### 发送Email
 
  	Uri uri = Uri.parse("mailto:android123@163.com");
  	Intent it = new Intent(Intent.ACTION_SENDTO, uri);
  	startActivity(it);
 
  	Intent it = new Intent(Intent.ACTION_SEND);
 	it.putExtra(Intent.EXTRA_EMAIL, "android123@163.com");
  	it.putExtra(Intent.EXTRA_TEXT, "android开发网测试");
  	it.setType("text/plain");
  	startActivity(Intent.createChooser(it, "选择一个Email客户端"));  
 
	Intent it=new Intent(Intent.ACTION_SEND);   
	String[] tos={"android123@163.com"};     //发送到 
	String[] ccs={"ophone123@163.com"};    //抄送
	it.putExtra(Intent.EXTRA_EMAIL, tos);   
	it.putExtra(Intent.EXTRA_CC, ccs);   
	it.putExtra(Intent.EXTRA_TEXT, "正文");   
	it.putExtra(Intent.EXTRA_SUBJECT, "标题");   
	it.setType("message/rfc822");    //编码类型
	startActivity(Intent.createChooser(it, "选择一个Email客户端"));
 
### Email添加附件
 
	Intent it = new Intent(Intent.ACTION_SEND);
  	it.putExtra(Intent.EXTRA_SUBJECT, "正文");
  	it.putExtra(Intent.EXTRA_STREAM, "file:///sdcard/nobody.mp3"); //附件为sd卡上的nobody MP3文件
  	sendIntent.setType("audio/mp3");
  	startActivity(Intent.createChooser(it, "选择一个Email客户端"));
 
### 播放多媒体
  
  	Intent it = new Intent(Intent.ACTION_VIEW);
  	Uri uri = Uri.parse("file:///sdcard/nobody.mp3");
  	it.setDataAndType(uri, "audio/mp3");
  	startActivity(it);
 
  	Uri uri = Uri.withAppendedPath(MediaStore.Audio.Media.INTERNAL_CONTENT_URI, "1"); //从系统内部的MediaProvider索引中调用播放
  	Intent it = new Intent(Intent.ACTION_VIEW, uri);
  	startActivity(it);  
 
### Uninstall卸载程序
 
  	Uri uri = Uri.fromParts("package", packageName, null); //packageName为包名，比如com.android123.apkInstaller
  	Intent it = new Intent(Intent.ACTION_DELETE, uri);
  	startActivity(it);

 
### 进入联系人界面
 
	Intent intent = new Intent();
	intent.setAction(Intent.ACTION_VIEW);
	intent.setData(People.CONTENT_URI);
	startActivity(intent);
 
查看某个联系人，当然这里是ACTION_VIEW，如果为选择并返回action改为ACTION\_PICK，当然处理intent时返回需要用到startActivityforResult
 
 	Uri personUri = ContentUris.withAppendedId(People.CONTENT_URI, ID);//最后的ID参数为联系人Provider中的数据库BaseID，即哪一行
 	Intent intent = new Intent();
 	intent.setAction(Intent.ACTION_VIEW);
 	intent.setData(personUri);
	startActivity(intent);
 
### 选择一个图片
 
	Intent intent = new Intent(Intent.ACTION_GET_CONTENT);  
	intent.addCategory(Intent.CATEGORY_OPENABLE);  
	intent.setType("image/*");//此处->audio/*对应选择音频 video/*对应选择视频
	startActivityForResult(intent, 0);
 
### 调用Android设备的照相机，并设置拍照后存放位置
 
 	Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);  
	intent.putExtra(MediaStore.EXTRA_OUTPUT, Uri.fromFile(new File(Environment.getExternalStorageDirectory().getAbsolutePath()+"/cwj", android123 + ".jpg"))); //存放位置为sdcard卡上cwj文件夹，文件名为android123.jpg格式
	startActivityForResult(intent, 0);
 
### 搜索指定package name在market上
 
	Uri uri = Uri.parse("market://search?q=pname:com.android123.cwj");   
	Intent intent = new Intent(Intent.ACTION_VIEW, uri);   
	startActivity(intent); 