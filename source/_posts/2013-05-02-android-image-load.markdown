---
layout: post
title: "Android异步加载图片"
date: 2013-05-02 20:02:50 +0800
comments: true
categories: android 
---

对于Android中的异步加载图片，自己总结了两种方式，如下：

1.<pre>
/** 
 * 异步读取图片，需要传递三个参数：Imageview imageView,String imagePath,int maxNumpixels  
 * @author Bryant 
 */  
public class AsyncLoadImage extends AsyncTask<Object, Object, Void> {  
    @Override  
    protected Void doInBackground(Object... params) {  

        try {  
            ImageView imageView=(ImageView) params[0];  
            String path=(String) params[1];  
            int maxNumOfPixels = (Integer)params[2];
            
            Bitmap bm = CompressPicture.compress(path,maxNumOfPixels);
   
            publishProgress(new Object[] {imageView, bm});  
        }catch (Exception e) {  
            e.printStackTrace();  
        }  
        
        return null;
    }  

    protected void onProgressUpdate(Object... progress) {  
        ImageView imageView = (ImageView) progress[0];  
        Bitmap bm = (Bitmap) progress[1];
        if(bm != null){
            imageView.setImageBitmap(bm); 
        }

    }  
}  
</pre>
 
2.

<pre>
/**
 * 图片异步加载
 * @author Bryant
 *
 */
public class AsyncImageLoader {
	private Map<String, SoftReference<Bitmap>> imageCache=new HashMap<String, SoftReference<Bitmap>>();
	private int maxNumPixels;
	
	public Bitmap loadBitmap(final String imagePath,int maxNumPixels,final ImageCallback callback){
		this.maxNumPixels = maxNumPixels;
		
		if(imageCache.containsKey(imagePath)){
			SoftReference<Bitmap> softReference=imageCache.get(imagePath);
			if(softReference.get()!=null){
				return softReference.get();
			}
		}
		final Handler handler=new Handler(){
			@Override
			public void handleMessage(Message msg) {
				callback.imageLoaded((Bitmap) msg.obj, imagePath);
			}
		};
		new Thread(){
			public void run() {
				Bitmap bitmap=loadImageFromPath(imagePath);
				imageCache.put(imagePath, new SoftReference<Bitmap>(bitmap));
				handler.sendMessage(handler.obtainMessage(0,bitmap));
			};
		}.start();
		return null;
	}
	
	protected Bitmap loadImageFromPath(String imagePath) {
		try {
			return CompressPicture.compress(imagePath, maxNumPixels);
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}

	public interface ImageCallback{
		public void imageLoaded(Bitmap imageBitmap,String imagePath);
	}
}
</pre>
 
经过实际使用，AsyncImageLoader的效率比较高，但是调用比较麻烦。
调用示例：
<pre>
private AsyncImageLoader imageLoader = new AsyncImageLoader();
imageLoader.loadBitmap(imagePath,maxNumPixels, new ImageCallback() {
	public void imageLoaded(Bitmap imageBitmap, String imagePath) {
		imageView.setImageBitmap(imageBitmap);
	}
}); 
</pre>
AsyncLoadImage调用较简单，调用示例如下：
<pre>
new AsyncLoadImage().execute(new Object[]{imageView,imagePath,maxNumPixels});
</pre>