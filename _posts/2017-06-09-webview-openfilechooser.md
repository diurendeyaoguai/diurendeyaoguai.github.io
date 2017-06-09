---
layout: post
title: WebView调用系统图库和拍照上传的问题
categories: WebView
description: WebView拍照和从图库里选择图片上传
keywords: WebView
---

## 需求

项目里有一个需求，需要使用h5调起本地图库和拍照。我给出的经验是使用jsbridge，通过js调用本地方法然后回传数据的方案。h5那边说是已经做完了，需要使用系统的方法来搞，好吧。。。
这样你们倒是简单极了

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"
        "http://www.w3.org/TR/html4/loose.dtd">
<html>
<body>
        <input id="fileImage" type="file" size="300" name="fileselect" accept="image/*" />
</body>
</html>
```
我们看起来也很简单的样子，毕竟浏览器里打开毫无压力。webview试一下吧，wtf...google一下吧，google告诉我“需要继承WebChromeCilent,重写WebChromeClient的onFileChooser方法”。
做完了，发现好像5.0,6.0都不行，是的就是那么坑，原来是攻城狮们在5.0以后的版本里弄了个新方法onShowFileChooser。大功告成之后可以愉快的打包扔给QA了，刚扔过去就给踢回来了，
依然不行。但是我刚刚明明调试通了啊，那再debug一遍，依然没问题。忽然想到是不是proguard的问题，毕竟很多自己确定可以但打包之后QA告诉你不可以的问题都是这货的问题，那就加一下吧
加完之后果然就解决了完美的不像实力派。
```java

public class MyWebChromeClient extends WebChromeClient{
    private Context context;
    private ValueCallback<Uri> mUploadMessage;
    private Uri mCapturedImageURI = null;
    private ValueCallback<Uri[]> mFilePathCallback;
    private String mCameraPhotoPath;
    private static final int INPUT_FILE_REQUEST_CODE = 1;
    private static final int FILECHOOSER_RESULTCODE = 1;
    
    
    
    public MyWebChromeClient(Context context) {
        this.context = context;
    }


    private File createImageFile() throws IOException {
        // Create an image file name
        String timeStamp = new SimpleDateFormat("yyyyMMdd_HHmmss").format(new Date());
        String imageFileName = "JPEG_" + timeStamp + "_";
        File storageDir = Environment.getExternalStoragePublicDirectory(
                Environment.DIRECTORY_PICTURES);
        File imageFile = File.createTempFile(
                imageFileName,  /* prefix */
                ".jpg",         /* suffix */
                storageDir      /* directory */
        );
        return imageFile;
    }



    // For Android 5.0
        public boolean onShowFileChooser(WebView view, final ValueCallback<Uri[]> filePath, WebChromeClient.FileChooserParams fileChooserParams) {
            //针对6.0请求权限的工具类
            PermissionCheckHelper.instance().requestPermissions(context, 0, permissions, permissionsMsg, new PermissionCheckHelper.PermissionCallbackListener() {
                @Override
                public void onPermissionCheckCallback(int code, String[] permissions, int[] grantResults) {
                    for (String permission : permissions) {
                        if (!PermissionCheckHelper.isPermissionGranted(context, permission)) {
                            return;
                        }
                    }
                    //todo


                    // Double check that we don't have any existing callbacks
                    if (mFilePathCallback != null) {
                        mFilePathCallback.onReceiveValue(null);
                    }
                    mFilePathCallback = filePath;

                    Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
                    if (takePictureIntent.resolveActivity(context.getPackageManager()) != null) {
                        // Create the File where the photo should go
                        File photoFile = null;
                        try {
                            photoFile = createImageFile();
                            takePictureIntent.putExtra("PhotoPath", mCameraPhotoPath);
                        } catch (IOException ex) {
                            // Error occurred while creating the File
                            //Log.e(Common.TAG, "Unable to create Image File", ex);
                        }

                        // Continue only if the File was successfully created
                        if (photoFile != null) {
                            mCameraPhotoPath = "file:" + photoFile.getAbsolutePath();
                            takePictureIntent.putExtra(MediaStore.EXTRA_OUTPUT,
                                    Uri.fromFile(photoFile));
                        } else {
                            takePictureIntent = null;
                        }
                    }

                    Intent contentSelectionIntent = new Intent(Intent.ACTION_GET_CONTENT);
                    contentSelectionIntent.addCategory(Intent.CATEGORY_OPENABLE);
                    contentSelectionIntent.setType("image/*");

                    Intent[] intentArray;
                    if (takePictureIntent != null) {
                        intentArray = new Intent[]{takePictureIntent};
                    } else {
                        intentArray = new Intent[0];
                    }

                    Intent chooserIntent = new Intent(Intent.ACTION_CHOOSER);
                    chooserIntent.putExtra(Intent.EXTRA_INTENT, contentSelectionIntent);
                    chooserIntent.putExtra(Intent.EXTRA_TITLE, "Image Chooser");
                    chooserIntent.putExtra(Intent.EXTRA_INITIAL_INTENTS, intentArray);

                    context.startActivityForResult(chooserIntent, INPUT_FILE_REQUEST_CODE);


                }
            });


            return true;

        }

        // openFileChooser for Android 3.0+
    public void openFileChooser(final ValueCallback<Uri> uploadMsg, final String acceptType) {

        

                mUploadMessage = uploadMsg;
                // Create AndroidExampleFolder at sdcard
                // Create AndroidExampleFolder at sdcard

                File imageStorageDir = new File(
                        Environment.getExternalStoragePublicDirectory(
                                Environment.DIRECTORY_PICTURES)
                        , "webcamera");

                if (!imageStorageDir.exists()) {
                    // Create AndroidExampleFolder at sdcard
                    imageStorageDir.mkdirs();
                }

                // Create camera captured image file path and name
                File file = new File(
                        imageStorageDir + File.separator + "IMG_"
                                + String.valueOf(System.currentTimeMillis())
                                + ".jpg");

                mCapturedImageURI = Uri.fromFile(file);

                // Camera capture image intent
                final Intent captureIntent = new Intent(
                        android.provider.MediaStore.ACTION_IMAGE_CAPTURE);

                captureIntent.putExtra(MediaStore.EXTRA_OUTPUT, mCapturedImageURI);

                Intent i = new Intent(Intent.ACTION_GET_CONTENT);
                i.addCategory(Intent.CATEGORY_OPENABLE);
                i.setType("image/*");

                // Create file chooser intent
                Intent chooserIntent = Intent.createChooser(i, "Image Chooser");

                // Set camera intent to file chooser
                chooserIntent.putExtra(Intent.EXTRA_INITIAL_INTENTS
                        , new Parcelable[]{captureIntent});

                // On select image call onActivityResult method of activity
                context.startActivityForResult(chooserIntent, FILECHOOSER_RESULTCODE);


    }

    // openFileChooser for Android < 3.0
    public void openFileChooser(ValueCallback<Uri> uploadMsg) {
        openFileChooser(uploadMsg, "");
    }

    //openFileChooser for other Android versions
    public void openFileChooser(ValueCallback<Uri> uploadMsg,
                                String acceptType,
                                String capture) {

        openFileChooser(uploadMsg, acceptType);
    }
}




```
在onActivity里做处理上传的方法

```java


@Override
    public void onActivityResult(int requestCode, int resultCode, Intent data) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {

            if (requestCode != INPUT_FILE_REQUEST_CODE || mFilePathCallback == null) {
                super.onActivityResult(requestCode, resultCode, data);
                return;
            }

            Uri[] results = null;

            // Check that the response is a good one
            if (resultCode == RESULT_OK) {
                if (data == null) {
                    // If there is not data, then we may have taken a photo
                    if (mCameraPhotoPath != null) {
                        results = new Uri[]{getPath(Uri.parse(mCameraPhotoPath))};
                    }
                } else {
                    String dataString = data.getDataString();
                    if (dataString != null) {
                        results = new Uri[]{getPath(Uri.parse(dataString))};
                    } else if (mCameraPhotoPath != null) {
                        results = new Uri[]{getPath(Uri.parse(mCameraPhotoPath))};
                    }
                }
            }

            mFilePathCallback.onReceiveValue(results);
            mFilePathCallback = null;

        } else if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.KITKAT) {
            if (requestCode != FILECHOOSER_RESULTCODE || mUploadMessage == null) {
                super.onActivityResult(requestCode, resultCode, data);
                return;
            }

            if (requestCode == FILECHOOSER_RESULTCODE) {

                if (null == this.mUploadMessage) {
                    return;

                }

                Uri result = null;

                try {
                    if (resultCode != RESULT_OK) {

                        result = null;

                    } else {

                        // retrieve from the private variable if the intent is null
                        result = data == null ? mCapturedImageURI : data.getData();
                    }
                } catch (Exception e) {
                    Toast.makeText(getActivity(), "activity :" + e,
                            Toast.LENGTH_LONG).show();
                }

                mUploadMessage.onReceiveValue(getPath(result));

                mUploadMessage = null;

            }
        }

        return;
    }



```
getPath()方法
```java
 
    private Uri getPath(Uri uri) {
        if (uri != null && !TextUtils.isEmpty(uri.toString()) && uri.toString().startsWith("file")) {
            String pathPic = Utility.getPath(getActivity(), uri);//这里讲sd卡路径转为receiver路径
            File cameraFile = new File(pathPic);
            if (cameraFile.exists()) {
                String media = null;
                try {
                    media = MediaStore.Images.Media.insertImage(getActivity().getContentResolver(), pathPic, "", "");
                    return Uri.parse(media);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }

        }
        return uri;
    }


```
是的，这样就完成了。	

## 实现
解决方法

1，android提供方法给js调用，选择图片、上传图片在android本地完成。
2，复写openFileChooser和onShowFileChooser方法

