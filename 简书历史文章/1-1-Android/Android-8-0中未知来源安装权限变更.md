##一、问题现象
在测试APK升级逻辑时，偶然发现在8.0系统的手机中，APK下载完就没有然后了，没有弹出安装界面，不执行安装逻辑。但是在8.0之前的版本中可以正常下载，正常弹起安装界面。

##二、问题分析

查阅相关资料发现，Android8.0中对于APK的安装做了如下调整：
* 将 设置--安全 中的 **允许安装未知来源应用** 取消了（由于国内手机系统的高度定制，该选择项的位置有差异）
* 在安装 APK 文件时新增 **未知来源安装权限**，即 `android.permission.REQUEST_INSTALL_PACKAGES`

也就是说，在Android 8.0(即Android O) 之前，设置 中的 允许安装未知来源 是针对所有APP的，只要开启了，那么所有的未知来源APP都可以安装。但是，8.0之后，将这个权限挪到了每一个APP内部，这样提高了手机的安全性，降低了流氓软件的安装概率。

参考资料: [Making it safer to get apps on Android O](https://android-developers.googleblog.com/2017/08/making-it-safer-to-get-apps-on-android-o.html "Making it safer to get apps on Android O")

##三、解决方案

### （1）、步骤1
按照上面参考资料中的说明，现在 AndroidMainfest.xml 清单文件中增加如下权限
```
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />
```

###（2）、步骤2
在上述参考资料中，有下面这么一段话：
>You can choose to pre-emptively direct your users to the **Install unknown apps** permission screen using the [ACTION_MANAGE_UNKNOWN_APP_SOURCES](https://developer.android.com/reference/android/provider/Settings.html#ACTION_MANAGE_UNKNOWN_APP_SOURCES) Intent action. 
You can also query the state of this permission using the PackageManager [canRequestPackageInstalls()](https://developer.android.com/reference/android/content/pm/PackageManager.html#canRequestPackageInstalls()) API.

上面这段话意思是说，
* 我们通过  [ACTION_MANAGE_UNKNOWN_APP_SOURCES](https://developer.android.com/reference/android/provider/Settings.html#ACTION_MANAGE_UNKNOWN_APP_SOURCES) 这个Action可以跳转到 未知来源安装设置界面，引导用户去开启这个选项。
* 我们可以通过PackageManager中的[canRequestPackageInstalls()](https://developer.android.com/reference/android/content/pm/PackageManager.html#canRequestPackageInstalls())来检测是否已经开启了未知来源安装权限。true 表示获取了权限，false 表示没有获取权限。为fasle 时，安装过程会被中断，无法跳转到安装界面。

所以，我们在下载完APK之后，可以按照下面的流程来处理代码：

![](https://upload-images.jianshu.io/upload_images/2551993-952c37f56b8723eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


具体示例代码如下：

* **下载逻辑省略，此处只列出 未知来源权限和安装 的处理逻辑**
* 下面的逻辑实在 WelcomeActivity中实现的，所以，可以直接使用 startActivityForResult 并在 onActivityResult中解析数据
```
 /**
     * 打开安装包
     */
    private void openAPKFile() {
        String mimeDefault = "application/vnd.android.package-archive";

        File apkFile = null;
        if (!TextUtils.isEmpty(mApkUri)) {
            //mApkUri是apk下载完成后在本地的存储路径
            apkFile = new File(Uri.parse(mApkUri).getPath());
        }
        if (apkFile == null) {
            return;
        }

        try {
            Intent intent = new Intent(Intent.ACTION_VIEW);
            intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            //兼容7.0
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
                intent.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
                //这里牵涉到7.0系统中URI读取的变更
                Uri contentUri = FileProvider.getUriForFile(mActivity, getPackageName() + ".fileprovider", apkFile);
                intent.setDataAndType(contentUri, mimeDefault);
                //兼容8.0
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                    boolean hasInstallPermission = getPackageManager().canRequestPackageInstalls();
                    if (!hasInstallPermission) {
                        startInstallPermissionSettingActivity();
                        return;
                    }
                }
            } else {
                intent.setDataAndType(Uri.fromFile(apkFile), mimeDefault);
            }
            if (getPackageManager().queryIntentActivities(intent, 0).size() > 0) {
                //如果APK安装界面存在，携带请求码跳转。使用forResult是为了处理用户 取消 安装的事件。外面这层判断理论上来说可以不要，但是由于国内的定制，这个加上还是比较保险的
                startActivityForResult(intent, 2);
            }
        } catch (Throwable e) {
            e.printStackTrace();
        }
    }

    /**
     * 跳转到设置-允许安装未知来源-页面
     */
    @RequiresApi(api = Build.VERSION_CODES.O)
    private void startInstallPermissionSettingActivity() {
        //后面跟上包名，可以直接跳转到对应APP的未知来源权限设置界面。使用startActivityForResult 是为了在关闭设置界面之后，获取用户的操作结果，然后根据结果做其他处理
        Intent intent = new Intent(Settings.ACTION_MANAGE_UNKNOWN_APP_SOURCES, Uri.parse("package:" + getPackageName()));
        startActivityForResult(intent, 1);
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (resultCode == RESULT_OK) {
            if (requestCode == 1) {
                openAPKFile();
            }
        } else {
            if (requestCode == 1) {
                //CnPeng 2018/8/2 下午4:31 8.0手机位置来源安装权限
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                    boolean hasInstallPermission = getPackageManager().canRequestPackageInstalls();
                    if (!hasInstallPermission) {
                        LogUtils.e(TAG, "没有赋予 未知来源安装权限");
                        showUnKnowResourceDialog();
                    }
                }
            } else if (requestCode == 2) {
                // CnPeng 2018/8/2 下午4:31 在安装页面中退出安装了
                LogUtils.e(TAG, "从安装页面回到欢迎页面--拒绝安装");

                showApkInstallDialog();
            }
        }
    }

    /**
     * 作者：CnPeng
     * 时间：2018/8/2 下午6:06
     * 功用：弹窗请安装APP的弹窗
     * 说明：8.0手机升级APK时获取了未知来源权限，并跳转到APK界面后，用户可能会选择取消安装，所以，再给一个弹窗
     */
    private void showApkInstallDialog() {
        final CustomAlertDialog installDialog = new CustomAlertDialog(mActivity);
        installDialog.setCancelable(false);
        DialogInstallApkBinding binding = DataBindingUtil.inflate(getLayoutInflater(), R.layout.dialog_install_apk, null, false);
        installDialog.setView(binding.getRoot());
        installDialog.show();

        binding.ivIKnowBt2.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //再次回到安装界面
                openAPKFile();
            }
        });

        binding.tvInstallNext.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                installDialog.dismiss();

                //CnPeng 2018/8/2 下午5:28  使用自定义方法关闭全部activity
                ActivitiesCollector.finishAll();
            }
        });
    }

    /**
     * 作者：CnPeng
     * 时间：2018/8/2 下午5:50
     * 功用：未知来源权限弹窗
     * 说明：8.0系统中升级APK时，如果跳转到了 未知来源权限设置界面，并且用户没用允许该权限，会弹出此窗口
     */
    private void showUnKnowResourceDialog() {
        final CustomAlertDialog alertDialog = new CustomAlertDialog(mActivity);
        alertDialog.setCancelable(false);

        DialogUnknowResourceBinding binding = DataBindingUtil.inflate(getLayoutInflater(), R.layout.dialog_unknow_resource, null, false);
        alertDialog.setView(binding.getRoot());
        alertDialog.show();

        binding.ivIKnowBt.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //兼容8.0
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                    boolean hasInstallPermission = getPackageManager().canRequestPackageInstalls();
                    if (!hasInstallPermission) {
                        startInstallPermissionSettingActivity();
                    }
                }

                alertDialog.dismiss();
            }
        });
    }
```

##四、总结
###（1）、注意
感谢评论中 @[D___Will](https://www.jianshu.com/u/321edc1ff4c9) 关于的 targetSdkVersion 提醒：
>**module 的 build.gradle 中 targetSdkVersion 不能低于 26**

* google文档中的原话如下：
An application must target Android O or higher and declare permission [Manifest.permission.REQUEST_INSTALL_PACKAGES](https://developer.android.google.cn/reference/android/Manifest.permission.html#REQUEST_INSTALL_PACKAGES)


###（2）、个人总结
在关注新版本特性时，不能只关注新控件，其他系统级的变更必须高度重视。这次的8.0安装权限变更就是一个教训啊！！

###（2）、参考资料附录
 [Making it safer to get apps on Android O](https://android-developers.googleblog.com/2017/08/making-it-safer-to-get-apps-on-android-o.html "Making it safer to get apps on Android O")

---
本文到此结束，谢谢观看！
**如有不足，敬请指正！**



