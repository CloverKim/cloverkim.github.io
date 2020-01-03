---
title: iOS开发 - Face ID & Touch ID
categories: iOS开发
tags:
- iOS
top: 100
copyright: ture
---

# 概述
&emsp;&emsp;Touch ID指纹识别是iPhone 5s开始增加的一项重大功能，在实际的用途有：快捷登录、指纹支付等。Face ID面容识别是iPhone X开始增加的，与Touch ID类似，都是属于生物验证的范畴。在我们的App使用Face ID或Touch ID时，所使用的方法都是同样的，不得不感谢一下苹果对开发者的照顾。需要注意的是Touch ID需要iOS 8以上，Face ID需要iOS 11以上。
<!-- more -->
# Face ID & Touch ID API介绍
官方API文档：[https://developer.apple.com/documentation/localauthentication](https://developer.apple.com/documentation/localauthentication)

## 两个主要方法：
```
func canEvaluatePolicy(_ policy: LAPolicy, error: NSErrorPointer) -> Bool
```
1. 作用：判断设备是否支持Touch ID / Face ID。
2. 参数：
- LAPolicy：验证方式，后面会介绍；
- NSErrorPointer：这个参数不能传nil，我们需要用到此方法的error值进行下一步的判断。

```
func evaluatePolicy(_ policy: LAPolicy, localizedReason: String, reply: @escaping (Bool, Error?) -> Void)
```
1. 作用：调用系统授权，弹出Touch ID指纹识别框 / Face ID面容识别框。
2. 参数：
- LAPolicy：鉴定方式；
- localizedReason：我们应用程序需要验证指纹/面容的原因，具体地方如下图所示。官方强调，这个参数为必须有值，且不能为空。不然App将会闪退，并提示：
```
localizedReason parameter is mandatory and the call will throw NSInvalidArgumentException if nil or empty string is specified.
```
![](http://pic.cloverkim.com/749c46aagy1fx7mizkbgij20bd07pt9w.jpg 'Touch ID')

![](http://pic.cloverkim.com/749c46aagy1fx7miznpcsj209108ngm2.jpg 'Face ID')

- reply：验证回调，包含是否验证通过的Bool值和错误信息。 我们在拿到验证结果后，进行判断并进行下一步的处理操作，成功跳转或者错误提示等。

## LAPolicy 验证方式（两种）
1. 生物识别：.deviceOwnerAuthenticationWithBiometrics
&emsp;&emsp;Touch ID：首先弹出指纹识别的弹窗，当第一次指纹验证失败时，弹框会变成两个按钮：其中一个是“取消”，另一个按钮可以自定义标题和点击事件，自定义标题的变量为：localizedFallbackTitle；自定义点击事件，需要在错误回调中，拿到错误码为：kLAErrorUserFallback时，进行我们的点击事件即可。当3次指纹验证失败时，验证弹框会消失，此时还可以继续呼出验证弹框进行验证，若接下来的两次指纹识别都验证错误的话，Touch ID会锁住。
&emsp;&emsp;Face ID：当设备支持Face ID时，会调用Face ID的验证弹框，需要注意的是，面容识别是错误一次时，需要点击“再次尝试面容ID”按钮，才能继续验证，若第2次再次错误时，会弹出自定义标题按钮，和Touch ID的相同。5次识别错误后，Face ID也会被锁住。

1. 生物识别 + 密码认证：.deviceOwnerAuthentication
&emsp;&emsp;Touch ID：如果3次识别错误后，则会弹出系统密码输入验证页面，需要输入设备的密码来解锁。如果此时取消，还有2次调用指纹识别进行验证，如果都失败的话，接下来每次调用识别，都是用系统密码进行验证。
&emsp;&emsp;Face ID：当连续5次验证错误，即5次需要输入密码时，Face ID会被锁住，无法使用，需要进行系统密码验证过后来能继续使用。

## iOS 11 新增的属性LABiometryType
```
public enum LABiometryType : Int {

    /// 此设备不支持生物识别.
    @available(iOS 11.2, *)
    case none

    /// The device does not support biometry.
    @available(iOS, introduced: 11.0, deprecated: 11.2, renamed: "LABiometryType.none")
    public static var LABiometryNone: LABiometryType { get }

    /// 此设备支持 Touch ID.
    case touchID

    /// 此设备支持 Face ID.
    case faceID
}
```
&emsp;&emsp;该属性是用于判断当前设备支持的验证类型，我们可以拿到该属性判断支持Touch ID还是Face ID，从而自定义提示文字或者是localizedReason的文字。
&emsp;对于LABiometryType，苹果给出的一段注释需要特别注意：只有当调用了canEvaluatePolicy方法，返回是true并且没有错误的情况下才会设置该值，错误是指之前所说的NSErrorPointer，因此canEvaluatePolicy的第二个error参数不要设置为nil。在调用canEvaluatePolicy方法之前，或者调用后有error的情况下，该属性均无任何意义的值。
&emsp;因为是iOS 11新增的，因此在使用时，需要先判断当前的系统。
```
#available(iOS 11.0, *)
```

## LAError
&emsp;&emsp;该错误是在canEvaluatePolicy方法失败时返回的一个错误，即传值进去的error，所有错误和相关说明如下：
```
typedef NS_ENUM(NSInteger, LAError)
{
    /// 身份验证不成功，因为用户无法提供有效的凭据
    LAErrorAuthenticationFailed = kLAErrorAuthenticationFailed,
    
    /// 身份验证被用户取消 (e.g. 点击了取消按钮).
    LAErrorUserCancel = kLAErrorUserCancel,
    
    /// 身份验证被取消了，因为用户点击了返回按钮 (输入密码).
    LAErrorUserFallback = kLAErrorUserFallback,
    
    /// 身份验证被系统取消了 (e.g. 另外一个应用到了前台/当前应用退到了后台).
    LAErrorSystemCancel = kLAErrorSystemCancel,
    
    /// 验证无法启动，因为设备没有设置密码.
    LAErrorPasscodeNotSet = kLAErrorPasscodeNotSet,

    /// 身份验证无法启动，因为Touch ID在该设备不可用.
    LAErrorTouchIDNotAvailable NS_ENUM_DEPRECATED(10_10, 10_13, 8_0, 11_0, "use LAErrorBiometryNotAvailable") = kLAErrorTouchIDNotAvailable,

    /// 身份验证无法启动，因为Touch ID没有录入指纹.
    LAErrorTouchIDNotEnrolled NS_ENUM_DEPRECATED(10_10, 10_13, 8_0, 11_0, "use LAErrorBiometryNotEnrolled") = kLAErrorTouchIDNotEnrolled,

    /// 身份验证不成功，因为有太多失败的Touch ID尝试
    /// Touch ID现在是锁着的，解锁Touch ID必须使用密码, e.g. 调用
    /// LAPolicyDeviceOwnerAuthenticationWithBiometrics 的时候，输入密码是必要条件.
    LAErrorTouchIDLockout NS_ENUM_DEPRECATED(10_11, 10_13, 9_0, 11_0, "use LAErrorBiometryLockout")
        __WATCHOS_DEPRECATED(3.0, 4.0, "use LAErrorBiometryLockout") __TVOS_DEPRECATED(10.0, 11.0, "use LAErrorBiometryLockout") = kLAErrorTouchIDLockout,

    /// 应用程序取消了身份验证 (e.g. 在进行身份验证时，调用了invalidate).
    LAErrorAppCancel NS_ENUM_AVAILABLE(10_11, 9_0) = kLAErrorAppCancel,

    /// LAContext 传递给这个调用时，已经失效.
    LAErrorInvalidContext NS_ENUM_AVAILABLE(10_11, 9_0) = kLAErrorInvalidContext,

    /// 身份验证无法启动，因为生物识别在当前设备上不可用.
    LAErrorBiometryNotAvailable NS_ENUM_AVAILABLE(10_13, 11_0) __WATCHOS_AVAILABLE(4.0) __TVOS_AVAILABLE(11.0) = kLAErrorBiometryNotAvailable,

    /// 身份验证无法启动，因为生物识别没有录入信息.
    LAErrorBiometryNotEnrolled NS_ENUM_AVAILABLE(10_13, 11_0) __WATCHOS_AVAILABLE(4.0) __TVOS_AVAILABLE(11.0) = kLAErrorBiometryNotEnrolled,

    /// 身份验证不成功，因为太多次的验证失败并且生物识别验证处理锁定状态，必须输入密码才能解锁.
    LAErrorBiometryLockout NS_ENUM_AVAILABLE(10_13, 11_0) __WATCHOS_AVAILABLE(4.0) __TVOS_AVAILABLE(11.0) = kLAErrorBiometryLockout,
    
    /// 身份验证失败, 因为需要显示的UI界面使用了interactionNotAllowed属性
    LAErrorNotInteractive API_AVAILABLE(macos(10.10), ios(8.0), watchos(3.0), tvos(10.0)) = kLAErrorNotInteractive,
} NS_ENUM_AVAILABLE(10_10, 8_0) __WATCHOS_AVAILABLE(3.0) __TVOS_AVAILABLE(10.0);
```

# 集成Face ID & Touch ID
- 导入对应的头文件
```
import LocalAuthentication
```
- 使用canEvaluatePolicy方法判断当前的设备是否支持Face ID / Touch ID，若返回true且error不为空时，调用evaluatePolicy方法进行验证，系统会根据支持Face ID还是Touch ID，分别显示不同的验证方式和对话框。
- 验证结果会在evaluatePolicy的回调中，我们拿到对应的回调结果进行相应的处理即可。
- 如果是支持Face ID，需要在plist文件中，添加NSFaceIDUsageDescription。虽然不添加，也不会导致闪退或者其他问题，但是苹果在API文档中，还是标明了需要添加该key。
> Applications should also supply NSFaceIDUsageDescription key in the 
> Info.plist. This key identifies a string value that contains a 
> message to be displayed to users when the app is trying to use Face 
> ID for the first time. Users can choose to allow or deny the use of 
> Face ID by the app before the first use or later in Face ID privacy 
> settings. When the use of Face ID is denied, evaluations will fail 
> with LAErrorBiometryNotAvailable.

- 简单的demo部分代码：
```
class LocalAuthenticationService: NSObject {
    
    /// 进行Face ID / Touch ID验证
    class func verifyAuthentication(with type: BiometricsType, success: @escaping () -> Void) {
        guard type != .none else {
            return
        }
        let context = LAContext()
        context.evaluatePolicy(.deviceOwnerAuthentication, localizedReason: type == .touchID ? "通过Home键验证已有手机指纹" : "面容ID") { (isSuccess, error) in
            if isSuccess {
                success()
            }
        }
    }
    
    /// 判断是否支持Face ID / Touch ID
    class func justSupportBiometricsType() -> BiometricsType {
        let context = LAContext()
        let error: NSErrorPointer = nil
        if context.canEvaluatePolicy(.deviceOwnerAuthentication, error: error) {
            guard error == nil else {
                return .none
            }
            if #available(iOS 11.0, *) {
                return context.biometryType == .faceID ? .faceID : .touchID
            } else {
                return .touchID
            }
        }
        return .none
    }
}
```
- github链接：[UnlockDemo](https://github.com/CloverKim/UnlockDemo)
## 运行效果图
![](http://pic.cloverkim.com/749c46aagy1fx7k6vlb9yg20op0dwe81.gif 'Touch ID')

![](http://pic.cloverkim.com/749c46aagy1fx7k1sjb6ag20op0dwb29.gif 'Face ID')

# 参考
- [csdn - mengkeer - iOS FaceID & TouchID](https://blog.csdn.net/zuoqianheng/article/details/80594389)
- [简书 - zuoqianheng- iOS集成FaceID或TouchID](https://www.jianshu.com/p/277ab9d45115)