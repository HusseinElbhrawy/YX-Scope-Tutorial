## **تلخيص عام عن yx_scope و yx_scope_flutter**

### **yx_scope**
- **إيه هو؟**: مكتبة Dart نقية لإدارة الـ Dependency Injection (DI) بطريقة آمنة وقت الـ compile (compile-safe) مع دعم قوي لمفهوم الـ **scopes** (نطاقات).
- **الهدف منه**: بدل ما تعتمد على Service Locator أو طرق static، يوفر طريقة ديناميكية ومنظمة لإدارة التبعيات (dependencies) مع ضمان إنك مش هتوصل لأخطاء وقت التشغيل (runtime errors).
- **المميزات الرئيسية**:
  - ما بيعتمدش على code generation.
  - دعم نطاقات (scopes) متداخلة بأي عمق.
  - حماية من الـ circular dependencies.
  - دعم الـ asynchronous dependencies (يعني تبعيات بتاخد وقت عشان تتحمل).
  - lifecycle واضح ومحدد لكل تبعية.
  - ما بيربطش الـ DI بالـ UI (الـ UI هو اللي بيتبع الـ DI، مش العكس).

### **yx_scope_flutter**
- **إيه هو؟**: مكتبة إضافية بتدمج **yx_scope** مع Flutter، عشان تقدر تستخدم الـ scopes جوا الـ widget tree بسهولة.
- **الهدف منه**: يخليك تتحكم في الـ DI containers داخل تطبيقات Flutter، مع دعم لعرض الـ widgets بناءً على حالة الـ scope (زي لو الـ scope لسه بيتحمّل).
- **المميزات**:
  - يوفر widgets زي `ScopeProvider` و `ScopeBuilder` عشان تدمج الـ scopes مع الـ UI.
  - يدعم إدارة الـ scopes بناءً على lifecycle بتاع الـ widgets.

---

## **ليه نستخدم yx_scope؟**
الفريموورك ده بيحل مشاكل زي:
- إدارة التبعيات في تطبيقات معقدة من غير ما تحتاج تعمل كود كتير زيادة.
- ضمان إن التبعيات تتعامل معاها بطريقة آمنة وما تسببش أخطاء وقت التشغيل.
- فصل الـ business logic عن الـ UI، بحيث الـ DI هو اللي بيحدد إزاي الـ UI هيشتغل.

---

## **المفاهيم الأساسية (Key Entities)**

قبل ما ندخل في الأمثلة، لازم نفهم 3 حاجات أساسية في **yx_scope**:
1. **Dep (Dependency)**: حاوية (container) بتحتوي على instance واحد لكائن معين. بمعنى إنك بتعرف تبعية (dependency) وبتخزنها في الـ Dep.
2. **ScopeContainer**: مجموعة من التبعيات مترابطة مع بعض، بتكون معزولة وليها lifecycle واحد (يعني بيتم إنشاؤها وتدميرها مع بعض).
3. **ScopeHolder**: الكائن اللي بيحتفظ بحالة الـ ScopeContainer (يعني بيقولك هل الـ scope موجود ولا لأ)، وبيتحكم في إنشاء وتدمير الـ scope.

---

## **أمثلة تدريجية**

هنبدأ من حاجات بسيطة جدًا ونتدرج لحد مستوى متقدم، مع كود واضح وشرح لكل خطوة.

### **مستوى 1: أساسيات إنشاء Scope (مثال بسيط جدًا)**

**السيناريو**: عندك تطبيق صغير عايز تدير فيه تبعيتين:
- `AppRouterDelegate`: بيتحكم في التنقل.
- `AppStateObserver`: بيراقب حالة التطبيق ومحتاج الـ `AppRouterDelegate`.

**الخطوات**:
1. أضف **yx_scope** للـ `pubspec.yaml`:
   ```yaml
   dependencies:
     yx_scope: ^1.0.0
   ```

2. إنشاء الـ `ScopeContainer` والـ `ScopeHolder`:
   ```dart
   // app_scope.dart
   import 'package:yx_scope/yx_scope.dart';

   // الكلاس ده هو الـ ScopeContainer اللي هيحتوي على التبعيات
   class AppScopeContainer extends ScopeContainer {
     // تعريف تبعية AppRouterDelegate
     late final routerDelegateDep = dep(() => AppRouterDelegate());

     // تعريف تبعية AppStateObserver اللي بتعتمد على AppRouterDelegate
     late final appStateObserverDep = dep(() => AppStateObserver(routerDelegateDep.get));
   }

   // الكلاس ده هو الـ ScopeHolder اللي هيتحكم في الـ ScopeContainer
   class AppScopeHolder extends ScopeHolder<AppScopeContainer> {
     @override
     AppScopeContainer createContainer() => AppScopeContainer();
   }

   // كائنات وهمية عشان المثال
   class AppRouterDelegate {
     void navigate() => print("Navigating...");
   }

   class AppStateObserver {
     final AppRouterDelegate router;
     AppStateObserver(this.router);
     void observe() => print("Observing app state...");
   }
   ```

3. استخدام الـ Scope في الـ `main`:
   ```dart
   void main() async {
     // إنشاء ScopeHolder
     final appScopeHolder = AppScopeHolder();
     // إنشاء الـ Scope (هنا بيتم تهيئة التبعيات)
     await appScopeHolder.create();

     // التحقق من وجود الـ Scope
     final appScope = appScopeHolder.scope;
     if (appScope != null) {
       // الوصول للتبعيات
       final observer = appScope.appStateObserverDep.get;
       observer.observe(); // طباعة: Observing app state...
       observer.router.navigate(); // طباعة: Navigating...
     }

     // تدمير الـ Scope
     await appScopeHolder.drop();
   }
   ```

**الشرح**:
- الـ `ScopeContainer` بيحتوي على تعريف التبعيات باستخدام `dep()`.
- الـ `ScopeHolder` بيتحكم في إنشاء الـ scope وتدميره باستخدام `create()` و `drop()`.
- الـ `appScopeHolder.scope` ممكن يكون `null` لو الـ scope لسه ما اتعملش أو لو اتدمّر، وده بيضمن إنك تشيك على وجوده قبل ما تستخدم التبعيات (compile-safe).

---

### **مستوى 2: إضافة Scope إلى Flutter (مثال متوسط)**

**السيناريو**: عايز تستخدم الـ scope جوا تطبيق Flutter، بحيث الـ UI يستخدم التبعيات الموجودة في الـ scope.

**الخطوات**:
1. أضف **yx_scope_flutter** للـ `pubspec.yaml`:
   ```yaml
   dependencies:
     yx_scope: ^1.0.0
     yx_scope_flutter: ^1.0.0
   ```

2. استخدام الـ `ScopeProvider` و `ScopeBuilder` في الـ widget tree:
   ```dart
   // main.dart
   import 'package:flutter/material.dart';
   import 'package:yx_scope/yx_scope.dart';
   import 'package:yx_scope_flutter/yx_scope_flutter.dart';
   import 'app_scope.dart';

   void main() {
     runApp(const App());
   }

   class App extends StatefulWidget {
     const App({super.key});

     @override
     State<App> createState() => _AppState();
   }

   class _AppState extends State<App> {
     final _appScopeHolder = AppScopeHolder();

     @override
     void initState() {
       super.initState();
       _appScopeHolder.create(); // إنشاء الـ scope لما الـ widget تتحمل
     }

     @override
     void dispose() {
       _appScopeHolder.drop(); // تدمير الـ scope لما الـ widget تتدمر
       super.dispose();
     }

     @override
     Widget build(BuildContext context) {
       return ScopeProvider(
         holder: _appScopeHolder,
         child: ScopeBuilder<AppScopeContainer>.withPlaceholder(
           builder: (context, appScope) {
             return MaterialApp(
               home: Scaffold(
                 appBar: AppBar(title: const Text('Yx Scope Demo')),
                 body: Center(
                   child: ElevatedButton(
                     onPressed: () {
                       appScope.appStateObserverDep.get.observe();
                     },
                     child: const Text('Observe State'),
                   ),
                 ),
               ),
             );
           },
           placeholder: const Center(child: CircularProgressIndicator()),
         ),
       );
     }
   }
   ```

**الشرح**:
- الـ `ScopeProvider` بيخلي الـ `ScopeHolder` متاح في الـ widget tree.
- الـ `ScopeBuilder` بيخليك تبني الـ UI بناءً على الـ scope الحالي، وبيوفر `placeholder` لو الـ scope لسه بيتحمّل.
- الـ lifecycle بتاع الـ scope بيتربط مع الـ widget lifecycle (يعني بيتم إنشاؤه في `initState` وتدميره في `dispose`).

---

### **مستوى 3: إدارة Scopes متداخلة (مثال متوسط-متقدم)**

**السيناريو**: عندك تطبيق فيه نطاق رئيسي (`AppScope`) ونطاق فرعي (`AccountScope`) بيظهر بس لما اليوزر يعمل login.

**الخطوات**:
1. إنشاء الـ `AccountScope`:
   ```dart
   // account_scope.dart
   import 'package:yx_scope/yx_scope.dart';

   class AccountScopeContainer extends ChildScopeContainer<AppScopeContainer> {
     AccountScopeContainer({required super.parent});

     late final userDep = dep(() => User());
     late final accountManagerDep = dep(() => AccountManager(userDep.get));
   }

   class AccountScopeHolder extends BaseChildScopeHolder<AccountScopeContainer, AppScopeContainer> {
     AccountScopeHolder(super.parent);

     @override
     AccountScopeContainer createContainer(AppScopeContainer parent) {
       return AccountScopeContainer(parent: parent);
     }
   }

   class User {
     String name = "Guest";
   }

   class AccountManager {
     final User user;
     AccountManager(this.user);
     void printUser() => print("User: ${user.name}");
   }
   ```

2. تعديل الـ `AppScopeContainer` عشان يدعم الـ `AccountScope`:
   ```dart
   // app_scope.dart
   class AppScopeContainer extends ScopeContainer {
     late final routerDelegateDep = dep(() => AppRouterDelegate());
     late final appStateObserverDep = dep(() => AppStateObserver(routerDelegateDep.get));

     // إضافة AccountScopeHolder كتبعية
     late final accountScopeHolderDep = dep(() => AccountScopeHolder(this));
   }
   ```

3. تعديل الـ UI عشان يدير الـ `AccountScope`:
   ```dart
   // main.dart
   class _AppState extends State<App> {
     final _appScopeHolder = AppScopeHolder();
     bool _isLoggedIn = false;

     @override
     void initState() {
       super.initState();
       _appScopeHolder.create();
     }

     @override
     void dispose() {
       _appScopeHolder.drop();
       super.dispose();
     }

     @override
     Widget build(BuildContext context) {
       return ScopeProvider(
         holder: _appScopeHolder,
         child: ScopeBuilder<AppScopeContainer>.withPlaceholder(
           builder: (context, appScope) {
             return MaterialApp(
               home: Scaffold(
                 appBar: AppBar(title: const Text('Yx Scope Demo')),
                 body: Center(
                   child: Column(
                     mainAxisAlignment: MainAxisAlignment.center,
                     children: [
                       if (_isLoggedIn)
                         ScopeBuilder<AccountScopeContainer>.withPlaceholder(
                           holder: appScope.accountScopeHolderDep.get,
                           builder: (context, accountScope) {
                             return ElevatedButton(
                               onPressed: () {
                                 accountScope.accountManagerDep.get.printUser();
                               },
                               child: const Text('Print User'),
                             );
                           },
                           placeholder: const CircularProgressIndicator(),
                         ),
                       ElevatedButton(
                         onPressed: () async {
                           if (_isLoggedIn) {
                             await appScope.accountScopeHolderDep.get.drop();
                           } else {
                             await appScope.accountScopeHolderDep.get.create();
                           }
                           setState(() => _isLoggedIn = !_isLoggedIn);
                         },
                         child: Text(_isLoggedIn ? 'Logout' : 'Login'),
                       ),
                     ],
                   ),
                 ),
               ),
             );
           },
           placeholder: const Center(child: CircularProgressIndicator()),
         ),
       );
     }
   }
   ```

**الشرح**:
- الـ `AccountScopeContainer` هو نطاق فرعي (child scope) بيعتمد على الـ `AppScopeContainer` كـ parent.
- الـ `accountScopeHolderDep` بيخليك تدير الـ `AccountScope` من خلال الـ `AppScope`.
- الـ UI بيظهر زرار "Print User" بس لو اليوزر عامل login (يعني لو الـ `AccountScope` موجود).
- لما اليوزر يعمل logout، الـ `AccountScope` بيتدمر، وده بيضمن إن التبعيات زي `User` و `AccountManager` تتخلّص منها.

---

### **مستوى 4: إدارة تبعيات غير متزامنة (Asynchronous Dependencies) (مثال متقدم)**

**السيناريو**: عندك تبعية زي `ApiClient` محتاجة تتحمّل بشكل غير متزامن (asynchronous) قبل ما تستخدمها.

**الخطوات**:
1. تعديل الـ `AppScopeContainer` عشان يدعم تبعية غير متزامنة:
   ```dart
   // app_scope.dart
   import 'package:yx_scope/yx_scope.dart';

   class AppScopeContainer extends ScopeContainer {
     late final routerDelegateDep = dep(() => AppRouterDelegate());

     // تبعية غير متزامنة
     late final apiClientDep = rawAsyncDep(
       () => ApiClient(),
       init: (client) async {
         await client.init();
         return client;
       },
       dispose: (client) async => client.dispose(),
     );

     late final appStateObserverDep = dep(() => AppStateObserver(apiClientDep.get));

     // تحديد ترتيب تهيئة التبعيات الغير متزامنة
     @override
     List<Set<AsyncDep>> get initializeQueue => [
           {apiClientDep}
         ];
   }

   class ApiClient {
     Future<void> init() async {
       await Future.delayed(const Duration(seconds: 1)); // محاكاة عملية تحميل
       print("ApiClient initialized");
     }

     void fetchData() => print("Fetching data...");

     Future<void> dispose() async {
       print("ApiClient disposed");
     }
   }

   class AppStateObserver {
     final ApiClient apiClient;
     AppStateObserver(this.apiClient);
     void observe() => apiClient.fetchData();
   }
   ```

2. استخدام الـ scope في الـ UI:
   ```dart
   // main.dart
   class _AppState extends State<App> {
     final _appScopeHolder = AppScopeHolder();

     @override
     void initState() {
       super.initState();
       _appScopeHolder.create();
     }

     @override
     void dispose() {
       _appScopeHolder.drop();
       super.dispose();
     }

     @override
     Widget build(BuildContext context) {
       return ScopeProvider(
         holder: _appScopeHolder,
         child: ScopeBuilder<AppScopeContainer>.withPlaceholder(
           builder: (context, appScope) {
             return MaterialApp(
               home: Scaffold(
                 appBar: AppBar(title: const Text('Async Demo')),
                 body: Center(
                   child: ElevatedButton(
                     onPressed: () {
                       appScope.appStateObserverDep.get.observe();
                     },
                     child: const Text('Fetch Data'),
                   ),
                 ),
               ),
             );
           },
           placeholder: const Center(child: CircularProgressIndicator()),
         ),
       );
     }
   }
   ```

**الشرح**:
- الـ `rawAsyncDep` بيخليك تعرف تبعية غير متزامنة مع دالة `init` و `dispose`.
- الـ `initializeQueue` بيحدد إن الـ `apiClientDep` لازم تتحمّل قبل أي حاجة تانية.
- الـ `ScopeBuilder` هيظهر الـ placeholder (زي الـ CircularProgressIndicator) لحد ما الـ `apiClientDep` يخلّص تحميل.

---

### **مستوى 5: استخدام Interfaces و ScopeModule (مثال متقدم جدًا)**

**السيناريو**: عايز تنظم الكود بشكل أفضل باستخدام interfaces عشان تقلل الـ coupling، وتستخدم `ScopeModule` عشان تفصل تبعيات مترابطة منطقيًا.

**الخطوات**:
1. إنشاء interface للـ `AppScope`:
   ```dart
   // app_scope.dart
   import 'package:yx_scope/yx_scope.dart';

   abstract class AppScope implements Scope {
     AppStateObserver get appStateObserver;
     AccountScopeHolder get accountScopeHolder;
   }

   class AppScopeContainer extends ScopeContainer implements AppScope {
     late final routerDelegateDep = dep(() => AppRouterDelegate());
     late final appStateObserverDep = dep(() => AppStateObserver(routerDelegateDep.get));
     late final accountScopeHolderDep = dep(() => AccountScopeHolder(this));

     @override
     AppStateObserver get appStateObserver => appStateObserverDep.get;

     @override
     AccountScopeHolder get accountScopeHolder => accountScopeHolderDep.get;

     // نقل التبعيات الخاصة بالـ routing إلى ScopeModule
     late final routingModule = RoutingModule(this);
   }

   class RoutingModule extends ScopeModule<AppScopeContainer> {
     RoutingModule(super.container);

     late final routerDelegateDep = dep(() => AppRouterDelegate());
     late final appStateObserverDep = dep(() => AppStateObserver(routerDelegateDep.get));
   }
   ```

2. إنشاء interface للـ `AccountScope`:
   ```dart
   // account_scope.dart
   abstract class AccountScope implements Scope {
     AccountManager get accountManager;
   }

   class AccountScopeContainer extends ChildScopeContainer<AppScopeContainer>
       implements AccountScope {
     AccountScopeContainer({required super.parent});

     late final userDep = dep(() => User());
     late final accountManagerDep = dep(() => AccountManager(userDep.get));

     @override
     AccountManager get accountManager => accountManagerDep.get;
   }

   class AccountScopeHolder
       extends BaseChildScopeHolder<AccountScope, AccountScopeContainer, AppScopeContainer> {
     AccountScopeHolder(super.parent);

     @override
     AccountScopeContainer createContainer(AppScopeContainer parent) {
       return AccountScopeContainer(parent: parent);
     }
   }
   ```

3. تعديل الـ UI عشان يستخدم الـ interfaces:
   ```dart
   // main.dart
   class _AppState extends State<App> {
     final _appScopeHolder = AppScopeHolder();
     bool _isLoggedIn = false;

     @override
     void initState() {
       super.initState();
       _appScopeHolder.create();
     }

     @override
     void dispose() {
       _appScopeHolder.drop();
       super.dispose();
     }

     @override
     Widget build(BuildContext context) {
       return ScopeProvider(
         holder: _appScopeHolder,
         child: ScopeBuilder<AppScope>.withPlaceholder(
           builder: (context, appScope) {
             return MaterialApp(
               home: Scaffold(
                 appBar: AppBar(title: const Text('Advanced Demo')),
                 body: Center(
                   child: Column(
                     mainAxisAlignment: MainAxisAlignment.center,
                     children: [
                       if (_isLoggedIn)
                         ScopeBuilder<AccountScope>.withPlaceholder(
                           holder: appScope.accountScopeHolder,
                           builder: (context, accountScope) {
                             return ElevatedButton(
                               onPressed: () {
                                 accountScope.accountManager.printUser();
                               },
                               child: const Text('Print User'),
                             );
                           },
                           placeholder: const CircularProgressIndicator(),
                         ),
                       ElevatedButton(
                         onPressed: () async {
                           if (_isLoggedIn) {
                             await appScope.accountScopeHolder.drop();
                           } else {
                             await appScope.accountScopeHolder.create();
                           }
                           setState(() => _isLoggedIn = !_isLoggedIn);
                         },
                         child: Text(_isLoggedIn ? 'Logout' : 'Login'),
                       ),
                     ],
                   ),
                 ),
               ),
             );
           },
           placeholder: const Center(child: CircularProgressIndicator()),
         ),
       );
     }
   }
   ```

**الشرح**:
- الـ interfaces (`AppScope` و `AccountScope`) بيخفوا تفاصيل الـ implementation ويخلوا الكود أقل coupling.
- الـ `ScopeModule` (`RoutingModule`) بيفصل التبعيات الخاصة بالـ routing عن باقي التبعيات، بحيث الكود يبقى أنظف وأسهل في الصيانة.
- الـ `ScopeBuilder` بيستخدم الـ interfaces مباشرة، فالـ UI ما بيعتمدش على الـ container نفسه.

---

## **تلخيص النقاط الرئيسية**

1. **البساطة**: **yx_scope** بيوفر طريقة سهلة وآمنة لإدارة التبعيات بدون code generation.
2. **المرونة**: تقدر تعمل scopes متداخلة بأي عمق، مع دعم تبعيات غير متزامنة.
3. **الأمان**: بفضل null-safety والـ linter (yx_scope_linter)، الكود بيبقى محمي من أخطاء وقت الكتابة.
4. **الدمج مع Flutter**: **yx_scope_flutter** بيخليك تدمج الـ scopes مع الـ widget tree بسهولة.
5. **التنظيم**: استخدام interfaces و `ScopeModule` بيخلي الكود أنظف وأسهل في التعديل لما التطبيق يكبر.

---

## **نصايح للاستخدام المتقدم**
- **استخدم interfaces دايمًا**: عشان تقلل الـ coupling وتخلي الكود أكثر قابلية للتغيير.
- **نظم الـ scopes بحكمة**: ما تعملش scope جديد إلا لو عندك lifecycle مختلف فعلًا.
- **استخدم ScopeModule للتبعيات المتشابهة**: لو عندك مجموعة تبعيات مترابطة منطقيًا، حطها في module.
- **اهتم بالـ linter**: أضف **yx_scope_linter** عشان يساعدك تلاقي الأخطاء بدري.

---

لو عايز توضيح أكتر عن أي جزء أو عايز مثال على سيناريو معين، قولي وأنا أعملك كود مخصص! 😄
