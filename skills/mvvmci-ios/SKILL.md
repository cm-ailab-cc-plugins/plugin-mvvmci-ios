---
name: mvvmci-ios
description: >
  Use when building iOS features, adding screens, creating coordinators/viewmodels/interactors,
  or reviewing iOS architecture decisions in MVVM-C+I projects
---

# MVVM-C+I iOS Architecture

iOS 專案架構規範：Coordinator + MVVM + Interactor。本 skill 規範架構分層原則，具體實作以當下專案 codebase 為準。

## When to Activate

- 新增功能畫面（需要建立 Coordinator / VC / VM / Interactor）
- 修改既有功能的架構結構
- Review 程式碼是否符合 MVVM-C+I 分層
- 使用者提到 coordinator、interactor、viewmodel 等架構關鍵字

**不適用：** 純 UI 調整、不涉及架構的 bug fix、Provider / Adapter / API 層工作（依專案而異）

---

## 架構核心原則

### 1. 單向依賴

```
AppCoordinator → Coordinator → ViewController → ViewModel → Interactor
```

- 上層知道下層，下層透過 protocol + weak reference 回呼上層
- Coordinator 是唯一知道具體型別的地方，負責組裝所有元件

### 2. 狀態所有權

> Interactor 介面可以用以下幾種形式暴露資料 / 方法（詳見後方「Interactor」段落），**per data / per method 依實際需求選**，多種形式可在同一個 Interactor 並存：
> - **Publisher 屬性**（`var xxxPublisher: Published<T>.Publisher { get }`）：需要持續推送給 UI 的狀態
> - **Async throws 方法**（`func xxx(...) async throws -> T`）：一次性查詢、CRUD action、衍生查詢
> - **Get property**（`var xxx: T { get }`）：需同步讀當下值
>
> 對於 publisher 暴露的資料，Interactor 是 source of truth；async method 則是 facade，呼叫一次拿一次。

- **Interactor** 是業務邏輯與資料的入口：對於 publisher 暴露的資料，它持有 source of truth、以 `@Published` 維護；對於 async method 暴露的查詢/動作，它只是 facade
- **ViewModel** 只儲存 UI 狀態（`@Published` 顯示狀態），**不自己持有 source of truth**；資料一律從 Interactor 同步而來：
  - 對應 publisher 資料：訂閱 publisher（ViewModel 可 `@Published` 一份顯示 cache，但 cache 完全由 publisher 驅動，ViewModel 不自己 fetch / 維護）；或讀 protocol 的 `get` property 同步取當下值
  - 對應 async method：在 ViewModel action 方法內 `await` 拿結果，再 assign 給 `@Published` 顯示資料
- **例外**：跟 UI 強相關、跟資料來源無關的「行為元件」可以擺在 ViewModel，例如 `AVProvider`（包 `AVPlayer` 播放音檔）
- **ViewController** 是容器，持有 ViewModel 供 View 觀察

### 3. 弱引用規則

- Coordinator 中 `weak var parentCoordinator`
- ViewModel 中 `weak var coordinator`（protocol 型別）
- 所有可能造成循環參考的地方用 `weak`

### 4. Protocol 注入

- Coordinator protocol、Interactor protocol 皆以 protocol 型別傳入
- 具體型別只在 Coordinator 組裝時出現
- 便於測試與解耦
- Interactor 實作 `init` 通常採 default-value DI（如 `init(repository: ItemRepository = .shared)`），測試時注入 mock 即可替換

### 5. Coordinator 不儲存額外物件

- **Coordinator 禁止把組裝出來的物件（Interactor、ViewModel、ViewController 等）存成自己的 property**
- Coordinator 只儲存 `Coordinator` protocol 要求的屬性（`parentCoordinator`、`childCoordinators`）以及自身的 `rootViewController`、`navigator`
- 組裝位置有兩種，依 init 是否接收 `navigator` 決定：
  - **`start()` 中組裝**：Coordinator 自己建立 `UINavigationController`（模式 A），組裝放在 `start()`
  - **`init` 中組裝**：Coordinator 從外部接收 `navigator`（模式 B），組裝可直接寫在 `init` 中
- 組裝完成後由各層自行持有（VC 持有 VM、VM 持有 Interactor），Coordinator 不再保留 reference
- 如果有特殊需求需要儲存額外物件，應明確說明原因
- **例外**：`AppCoordinator` 可儲存全域 context（如 `window`），因為它管理 app 層級的生命週期

**命名慣例：**
- Coordinator protocol：`{功能名}CoordinatorProtocol`（如 `HomeCoordinatorProtocol`、`TabCoordinatorProtocol`）
- Coordinator 實作：`{功能名}Coordinator`（如 `HomeCoordinator`、`TabCoordinator`）
- Interactor protocol：`{功能名}Interactor`（如 `HomeInteractor`、`ItemListInteractor`）
- Interactor 實作：`{功能名}InteractorImpl`（如 `HomeInteractorImpl`、`ItemListInteractorImpl`）
- ViewModel：`{功能名}ViewModel`（如 `HomeViewModel`）
- ViewController：`{功能名}ViewController`（如 `HomeViewController`）

### 6. Publisher 轉發鏈

> 本規則適用於 Interactor 用 publisher 暴露的資料。Async method 走 async / await 直接 return，沒有 publisher 轉發鏈。

```
Interactor(@Published) → ViewModel(subscribe + transform) → View(@ObservedObject)
```

- ViewModel 在 `init` 中直接訂閱 Interactor 的 Publisher
- 使用 `.receive(on: DispatchQueue.main)` 確保 UI 執行緒

### 7. 導航統一由 Coordinator 處理

- 所有 `push` / `present` 都必須透過 Coordinator 呼叫；View / ViewModel 不直接接觸 `UINavigationController` 或 `present(_:)`
- ViewModel 透過 Coordinator protocol 觸發導航（如 `coordinator?.showDetail(id:)`）
- **`pop`** 由 Coordinator 處理（透過 `navigator`）
- **`dismiss`** 分兩種情況：
  - 一般 modal 畫面（非透過 `presentCoordinator()` 推出）：由該 VC 或 View 自己 dismiss，Coordinator 不接 dismiss callback（避免反向傳遞）
  - 透過 `presentCoordinator()` 推出的 child coordinator：必須由父 Coordinator 呼叫 `dismissCoordinator(_:animated:)` 回收（否則 `childCoordinators` 會殘留）

### 8. 檔案內容排列順序

每個 Swift 檔案內的成員按以下順序排列：

1. **Nested types**（enum、struct 等內嵌型別） — 如 `TabCoordinator` 內的 `enum Tab`
2. **變數**（properties） — `@Published`、`weak var`、`private var`、`let` 等
3. **`init`** — 初始化方法
4. **func** — 方法（public/internal 在前，private 在後）

---

## 各層規範與範例

### AppCoordinator

App 的根 Coordinator，管理全域導航流程與頂層 Coordinator 的切換。

**職責：**
- 持有 `window`，負責設定與替換 `rootViewController`
- 切換流程時先清除 `childCoordinators`，再建立新的子 Coordinator

**啟動流程：**

1. Main.storyboard 建立 `UIWindow` 和預設 ViewController（作為啟動/過渡畫面）
2. SceneDelegate 用 `window` 初始化 AppCoordinator
3. 預設 ViewController 透過 SceneDelegate 取得 AppCoordinator 引用，根據條件決定進入哪個流程

```swift
// SceneDelegate
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
    var appCoordinator: AppCoordinator?

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
        guard let _ = (scene as? UIWindowScene) else { return }
        appCoordinator = AppCoordinator(window: window)
    }
}

// AppCoordinator
class AppCoordinator: Coordinator {
    weak var parentCoordinator: Coordinator?
    var childCoordinators: [Coordinator] = []
    var rootViewController: UIViewController

    private let window: UIWindow?

    init(window: UIWindow?) {
        self.window = window
        self.rootViewController = window?.rootViewController ?? .init()
    }

    func start() {
        // 全域設定（如 NavigationBar 外觀）
    }

    // 切換到某個流程
    func showSomeFlow() {
        childCoordinators.removeAll()
        let coordinator = SomeCoordinator()
        addChildCoordinator(coordinator)
        window?.rootViewController = coordinator.rootViewController
        coordinator.start()
    }
}

// 預設 ViewController（storyboard 初始畫面）
class ViewController: UIViewController {
    private weak var appCoordinator: AppCoordinator?

    override func viewDidLoad() {
        super.viewDidLoad()
        appCoordinator = (UIApplication.shared.connectedScenes.first?.delegate as? SceneDelegate)?.appCoordinator
        appCoordinator?.start()
        determineFlow()
    }

    private func determineFlow() {
        Task {
            // 執行非同步判斷（如自動登入、取得使用者狀態等）
            await MainActor.run {
                // appCoordinator?.showOnboarding()
                // appCoordinator?.showMain()
            }
        }
    }
}
```

**子 Coordinator 反向觸發流程切換：**

子 Coordinator 不直接引用 AppCoordinator，而是透過自己的 protocol 定義方法，實作中向上傳遞：

```swift
// 子 Coordinator 的 protocol
protocol SomeCoordinatorProtocol: AnyObject {
    func switchToAnotherFlow()
}

// 子 Coordinator 實作
class SomeCoordinator: NavigationCoordinator, SomeCoordinatorProtocol {
    func switchToAnotherFlow() {
        if let appCoordinator = parentCoordinator as? AppCoordinator {
            appCoordinator.showAnotherFlow()
        }
    }
}

// ViewModel 中觸發
coordinator?.switchToAnotherFlow()
```

**新專案設定要點：**
- Info.plist 設定 `UISceneStoryboardFile` 指向 Main.storyboard
- Main.storyboard 放一個預設 ViewController（啟動過渡用，同時負責路由判斷）
- SceneDelegate 持有 `var appCoordinator: AppCoordinator?`

**專案目錄結構：** 基礎檔案扁平放在 target 根目錄，其餘按架構層分資料夾：

```
PlayerDemo/              ← target 根目錄
├── AppDelegate.swift        ← 扁平
├── SceneDelegate.swift      ← 扁平
├── ViewController.swift     ← 扁平（storyboard 預設 VC）
├── AppCoordinator.swift     ← 扁平
├── Info.plist               ← 扁平
├── Assets.xcassets/         ← 扁平
├── Base.lproj/              ← 扁平
│   ├── Main.storyboard
│   └── LaunchScreen.storyboard
├── Coordinator/             ← 各功能 Coordinator
│   ├── Coordinator/         ← 基礎設施四個檔案
│   │   ├── Coordinator.swift
│   │   ├── PresentationCoordinator.swift
│   │   ├── NavigationCoordinator.swift
│   │   └── Navigator.swift
│   ├── MainCoordinator.swift
│   └── PlayerCoordinator.swift
├── ViewController/          ← 各功能 VC
│   ├── AudioListVC.swift
│   └── PlayerVC.swift
├── ViewModel/               ← 各功能 VM
│   ├── AudioListVM.swift
│   └── PlayerVM.swift
├── Interactor/              ← 各功能 Interactor
│   ├── AudioListInteractor.swift
│   └── PlayerInteractor.swift
├── View/                    ← SwiftUI View
│   ├── AudioListView.swift
│   └── PlayerView.swift
├── Models/                  ← 資料模型
└── Provider/                ← Provider / Adapter 層
```

### Coordinator（功能層級）

每個功能模組一個 Coordinator，負責組裝元件與管理導航。

**職責：**
- 定義 protocol 供 ViewModel 弱引用
- 建立並組裝 ViewController + ViewModel + Interactor
- 透過 `navigator` push，或以自身 `rootViewController` 作為 presenting controller 呼叫 `present(...)` 彈出 modal（基礎設施提供 `pushCoordinator()` / `presentCoordinator()` 包好 child 生命週期管理）
- **pop** 由 Coordinator 處理（透過 `navigator`）
- **dismiss** 分兩種情況：
  - 一般 modal 畫面（非透過 `presentCoordinator()` 推出）：由該 VC 或 View 自行 dismiss
  - 透過 `presentCoordinator()` 推出的 child coordinator：由父 Coordinator 呼叫 `dismissCoordinator(_:animated:)` 回收

**方法命名規則：**

Coordinator 的方法名稱必須具體描述「要執行的動作」，而非描述「觸發的時機」：

| Bad | Good | 原因 |
|-----|------|------|
| `didCreate()` | `showDetail(id:)` | 應描述做什麼，不是什麼時候觸發 |
| `onComplete()` | `showMain()` | 同上 |
| `handleTap()` | `showProfile(userID:)` | Coordinator 不處理 UI 事件，只處理導航 |
| `next()` | `showPayment()` | 具體說明導航目標 |

常見的動詞前綴：`show`（跳轉畫面）、`present`（modal 彈出）、`switch`（切換流程）

**參數規則：**

> 本規則只規範 Coordinator 的**導航方法**（`func showXxx`、`func presentXxx`）；Coordinator 本身的 `init` 不在此限——模式 B 的 `init(interactor:, navigator:)` 接 Interactor 是允許的。

導航方法**傳遞資料物件，不傳遞 Interactor 本身**：

| Bad | Good | 原因 |
|-----|------|------|
| `func showDetail(interactor: DetailInteractor)` | `func showDetail(itemID: String)` | Interactor 由方法內部建立，呼叫端不需要知道 Interactor 存在 |

在方法內 init Interactor，再注入下一個 Coordinator：

```swift
func showDetail(itemID: String) {
    let interactor = DetailInteractorImpl(itemID: itemID)
    let coordinator = DetailCoordinator(interactor: interactor, navigator: navigator)
    pushCoordinator(coordinator, animated: true)
}
```

**Callback 例外：**

導航方法預設不收 callback（結果應透過 Interactor 狀態回傳）。但確實有需要回傳系統元件結果的場景（如 `UIActivityViewController` 的分享結果），可以接 `@escaping` closure：

```swift
func showShare(items: [Any], onComplete: @escaping (String, Bool) -> Void) {
    rootViewController.presentActivity(items: items) { type, complete in
        onComplete(type, complete)
    }
}
```

**兩種 Coordinator init 模式：**

**模式 A — 自己建立 NavigationController（根 Coordinator）：**

Coordinator 自己建立 `UINavigationController` 作為 `rootViewController`，在 `start()` 中組裝 VC/VM/Interactor。適用於作為導航起點的 Coordinator（如 TabCoordinator 內的各 tab root）。

```swift
protocol ItemCoordinatorProtocol: AnyObject {
    func showDetail(id: String)
}

class ItemCoordinator: NavigationCoordinator, ItemCoordinatorProtocol {
    var rootViewController: UINavigationController
    var navigator: NavigatorType
    weak var parentCoordinator: Coordinator?
    var childCoordinators: [Coordinator] = []

    init() {
        let navigationController = UINavigationController()
        self.navigator = Navigator(navigationController: navigationController)
        self.rootViewController = navigationController
    }

    func start() {
        let interactor = ItemListInteractorImpl()
        let viewModel = ItemListViewModel(coordinator: self, interactor: interactor)
        let viewController = ItemListViewController(viewModel: viewModel)
        rootViewController.setViewControllers([viewController], animated: false)
    }

    func showDetail(id: String) {
        let interactor = ItemDetailInteractorImpl(itemID: id)
        let detailCoordinator = ItemDetailCoordinator(interactor: interactor, navigator: navigator)
        pushCoordinator(detailCoordinator, animated: true)
    }
}
```

**模式 B — 接收外部 Navigator（子 Coordinator）：**

Coordinator 從父層接收 `navigator`，在 `init` 中就完成 VC/VM/Interactor 的組裝，`start()` 留空。適用於被 `pushCoordinator()` 推入的子 Coordinator，因為 push 時基礎設施需要存取 `rootViewController`，所以必須在 `init` 階段就準備好。

注意：因為 `init` 中 `self` 尚未完成初始化，ViewModel 的 coordinator 參數先傳 `nil`，init 結束後再補上 `self`。

```swift
protocol ItemDetailCoordinatorProtocol: AnyObject {
    func showRelated(id: String)
}

class ItemDetailCoordinator: NavigationCoordinator, ItemDetailCoordinatorProtocol {
    weak var parentCoordinator: Coordinator?
    var childCoordinators: [Coordinator] = []
    let rootViewController: UIViewController
    let navigator: NavigatorType

    init(interactor: ItemDetailInteractor, navigator: NavigatorType) {
        self.navigator = navigator
        let viewModel = ItemDetailViewModel(coordinator: nil, interactor: interactor)
        let vc = ItemDetailViewController(viewModel: viewModel)
        vc.hidesBottomBarWhenPushed = true
        self.rootViewController = vc
        viewModel.coordinator = self
    }

    func start() {
    }

    func showRelated(id: String) {
        let interactor = ItemDetailInteractorImpl(itemID: id)
        let relatedCoordinator = ItemDetailCoordinator(interactor: interactor, navigator: navigator)
        pushCoordinator(relatedCoordinator, animated: true)
    }
}
```


**TabBar 架構（含 UITabBarController）：**

當主畫面使用 TabBar 時，需要一個 TabCoordinator 管理多個 tab 的子 Coordinator。與一般 Coordinator 的差異：

- TabCoordinator 持有 `UITabBarController`，每個 tab 對應一個子 Coordinator
- 在 `start()` 中建立所有 tab 的子 Coordinator，將各自的 `rootViewController` 設為 tabController 的 `viewControllers`
- TabCoordinator 自己的 `rootViewController` 是 `UINavigationController`，tabController 被 push 進去（方便整頁跳轉時隱藏 tabBar）

```swift
protocol TabCoordinatorProtocol: AnyObject {
    func showFirstTab()
    func showSecondTab()
    func switchToAnotherFlow()
}

class TabCoordinator: NavigationCoordinator, TabCoordinatorProtocol {

    enum Tab: Int {
        case first
        case second

        var title: String {
            switch self {
            case .first: return "首頁"
            case .second: return "設定"
            }
        }

        var image: UIImage? {
            switch self {
            case .first: return UIImage(systemName: "house")
            case .second: return UIImage(systemName: "gearshape")
            }
        }

        var selectedImage: UIImage? {
            switch self {
            case .first: return UIImage(systemName: "house.fill")
            case .second: return UIImage(systemName: "gearshape.fill")
            }
        }
    }

    private lazy var tabVC: UITabBarController = {
        let vc = TabViewController(viewModel: TabViewModel(coordinator: self))
        return vc
    }()

    weak var parentCoordinator: Coordinator?
    var childCoordinators: [Coordinator] = []
    let rootViewController: UIViewController
    let navigator: NavigatorType

    init() {
        let navigationController = UINavigationController()
        navigationController.navigationBar.isHidden = true
        self.navigator = Navigator(navigationController: navigationController)
        self.rootViewController = navigationController
    }

    func start() {
        // 建立各 tab 的子 Coordinator
        let firstCoordinator = FirstCoordinator()
        addChildCoordinator(firstCoordinator)
        firstCoordinator.start()

        let secondCoordinator = SecondCoordinator()
        addChildCoordinator(secondCoordinator)
        secondCoordinator.start()

        // 將子 Coordinator 的 rootViewController 設為 tab 頁面
        tabVC.viewControllers = [
            firstCoordinator.rootViewController,
            secondCoordinator.rootViewController
        ]

        tabVC.hidesBottomBarWhenPushed = true
        navigator.push(tabVC, animated: true)
    }

    func showFirstTab() { tabVC.selectedIndex = Tab.first.rawValue }
    func showSecondTab() { tabVC.selectedIndex = Tab.second.rawValue }

    // 反向觸發 AppCoordinator 切換流程
    func switchToAnotherFlow() {
        if let appCoordinator = parentCoordinator as? AppCoordinator {
            appCoordinator.showSomeFlow()
        }
    }
}
```

**關鍵差異：**
- 一般 Coordinator 的 `start()` 只建立一組 VC/VM/Interactor
- TabCoordinator 的 `start()` 建立多個子 Coordinator，每個子 Coordinator 各自管理自己的畫面
- tabController 被 push 進 NavigationController，需要全螢幕跳轉時（如 detail 頁）可以從 navigator push 新畫面，tabBar 會自動隱藏


**共用 Navigator 的子 Coordinator：**

當子 Coordinator 需要 push 進父 Coordinator 的同一個 navigation stack 時（例如從列表頁 push 進詳情頁），子 Coordinator 接收父 Coordinator 的 `navigator`，在 `init` 中完成所有組裝，`start()` 為空。

```swift
protocol DetailCoordinatorProtocol: AnyObject {
    // 如有 detail 後續導航方法，宣告於此
}

class DetailCoordinator: NavigationCoordinator, DetailCoordinatorProtocol {
    weak var parentCoordinator: Coordinator?
    var childCoordinators: [Coordinator] = []
    let rootViewController: UIViewController
    let navigator: NavigatorType

    init(interactor: DetailInteractor, navigator: NavigatorType) {
        self.navigator = navigator
        let viewModel = DetailViewModel(coordinator: nil, interactor: interactor)
        let vc = DetailViewController(viewModel: viewModel)
        vc.hidesBottomBarWhenPushed = true
        self.rootViewController = vc
        viewModel.coordinator = self
    }

    func start() {}
}

// 父 Coordinator 中使用
func showDetail(itemID: String) {
    let interactor = DetailInteractorImpl(itemID: itemID)
    let coordinator = DetailCoordinator(interactor: interactor, navigator: navigator)
    pushCoordinator(coordinator, animated: true)
}
```

**子畫面實作三條路怎麼選：**

| 條件 | 自建 NavigationController | 共用 navigator + 獨立 child Coordinator | 共用 navigator + 父多 protocol |
|------|--------------------------|----------------------------------------|--------------------------------|
| 需要自己的 navigation stack | ✅ | ❌ | ❌ |
| 屬於父的 navigation stack 延伸 | ❌ | ✅ | ✅ |
| 子畫面有獨立子流程 / 會 push 自己的 child chain | ❌ | ✅ | ❌ |
| 子畫面是輕量單畫面（push / pop / present 即可） | ❌ | 可以但較重 | ✅ |
| 子畫面要被多入口 reuse | ✅（獨立流程） | ✅ | ✅（protocol 共用） |

**典型場景：**
- **自建 NavigationController**：Tab 的根 Coordinator、獨立流程（如 Onboarding）— 需要自己的 `UINavigationController`
- **共用 navigator + 獨立 child Coordinator**：同一 navigation stack 內、但子畫面有自己的子流程或 child chain — 接收父 Coordinator 的 `navigator`，`init` 中組裝完畢，`start()` 為空（見「共用 Navigator 的子 Coordinator」）
- **共用 navigator + 父多 protocol**：父功能流程包含多個輕量子畫面，不另建 Coordinator class，由父 Coordinator 直接實作各子畫面的 `CoordinatorProtocol`（見下方「多 protocol 實作」）

**多 protocol 實作（一個 Coordinator 服務多個子畫面）：**

當 Coordinator 管理的功能流程包含多個輕量子畫面（VC + VM 但不需要獨立 navigation flow），這些子畫面的 ViewModel 都需要呼叫導航時：
- **不另建** 每個子畫面的 Coordinator class
- 每個子畫面只有 protocol 檔（一個 `<Child>CoordinatorProtocol`），沒有對應的 Coordinator class
- **父 Coordinator class 同時遵從主 protocol + 所有子畫面 protocol**
- 子畫面 ViewModel init 時收 `coordinator: <Child>CoordinatorProtocol`，父 Coordinator 傳 `self` 即可

```swift
// 子畫面 A 的 protocol（檔名 ChildACoordinator.swift，但只含 protocol 宣告，無 class）
protocol ChildACoordinatorProtocol: AnyObject {
    func doSomething()
}

// 子畫面 B 的 protocol（檔名 ChildBCoordinator.swift）
protocol ChildBCoordinatorProtocol: AnyObject {
    func pop()
}

// 主功能 protocol
protocol ParentCoordinatorProtocol: AnyObject {
    func showChildA()
    func showChildB()
}

// 父 Coordinator class —— 同時實作主 protocol 與多個子畫面 protocol
class ParentCoordinator: NavigationCoordinator,
                          ParentCoordinatorProtocol,
                          ChildACoordinatorProtocol {

    weak var parentCoordinator: Coordinator?
    var childCoordinators: [Coordinator] = []
    let rootViewController: UIViewController
    let navigator: NavigatorType

    init(navigator: NavigatorType, interactor: ParentInteractor) {
        self.navigator = navigator
        let viewModel = ParentViewModel(coordinator: nil, interactor: interactor)
        self.rootViewController = ParentViewController(viewModel: viewModel)
        viewModel.coordinator = self
    }

    func start() {}

    // ParentCoordinatorProtocol
    func showChildA() {
        // 注意：coordinator: self —— self 因為實作了 ChildACoordinatorProtocol 自然 conform
        let viewModel = ChildAViewModel(coordinator: self, interactor: ChildAInteractorImpl())
        navigator.push(ChildAViewController(viewModel: viewModel), animated: true)
    }

    func showChildB() {
        let viewModel = ChildBViewModel(coordinator: self, interactor: ChildBInteractorImpl())
        navigator.push(ChildBViewController(viewModel: viewModel), animated: true)
    }

    // ChildACoordinatorProtocol
    func doSomething() {
        // ...
    }
}

// 概念獨立的通用方法（如 pop）用 extension 分離提升可讀性
extension ParentCoordinator: ChildBCoordinatorProtocol {
    func pop() {
        navigator.popViewController(animated: true)
    }
}
```

**子畫面 protocol 可被多個父 Coordinator 共用：**

如果同一個子畫面會從多個入口進入（例如 A 頁與 B 頁都能 push 進同一個 child 頁面），該子畫面的 `CoordinatorProtocol` 可以被多個父 Coordinator 同時實作。每個父 Coordinator 自己 push 該畫面、自己 conform protocol 處理那畫面內的導航事件 — 子畫面 protocol 本身不綁定特定父。

```swift
class FlowACoordinator: NavigationCoordinator, FlowACoordinatorProtocol, SharedChildCoordinatorProtocol { ... }
class FlowBCoordinator: NavigationCoordinator, FlowBCoordinatorProtocol, SharedChildCoordinatorProtocol { ... }
```

**核心適用情境：**
- 子畫面屬於同一個功能流程的延伸，不需要獨立 navigation stack
- 子畫面 ViewModel 只需要簡單的 push / pop / present / dismiss 等動作

**進階延伸：** 同一份子畫面 `CoordinatorProtocol` 可被多個父 Coordinator 同時實作（前提是各父都用本模式）— 適合通用子畫面（如「分享」「VIP 升級」「設定提醒」等會從多入口進入的頁面）。

**不適合多 protocol 的情況（應該獨立成 child Coordinator）：**
- 子畫面是獨立流程：有自己的 navigation stack、會 push 自己的 child Coordinator chain、或有獨立生命週期管理需求
- 多個 protocol 方法集合過大，導致父 Coordinator class 失焦

**檔案結構慣例：**
- 每個子畫面有自己的 protocol 檔：`<Child>CoordinatorProtocol` 宣告在 `<Child>Coordinator.swift` 中
- 該檔只有 protocol 宣告，無 Coordinator class
- 父 Coordinator class 跟著主功能命名

**實作要點：**
- 多 protocol 的方法可全寫在 class body 內，或按概念分群用 `extension` 分離
- 子畫面 ViewModel init 時收的 coordinator 型別是 `<Child>CoordinatorProtocol`（不是父的具體 class），確保子畫面對父實作解耦
- 若不同 protocol 有同名方法（如多個 protocol 都宣告 `pop()`），父 Coordinator 一份實作可同時 satisfy 多個 protocol

---

**Coordinator 基礎設施：** 以下四個檔案是基礎設施模板。新專案如果沒有這些檔案，直接複製到專案中。

#### Coordinator.swift

```swift
import UIKit

protocol Coordinator: AnyObject {
    var parentCoordinator: Coordinator? { get set }
    var childCoordinators: [Coordinator] { get set }

    func start()
}

extension Coordinator {
    
    func addChildCoordinator(_ childCoordinator: Coordinator) {
        childCoordinators.append(childCoordinator)
        childCoordinator.parentCoordinator = self
    }
    
    func removeChildCoordinator(_ childCoordinator: Coordinator) {
        childCoordinators.removeAll {$0 === childCoordinator}
    }
    
}
```

#### PresentationCoordinator.swift

```swift
import UIKit

// swiftlint:disable all
protocol _PresentationCoordinator: Coordinator {
    
    var _rootViewController: UIViewController { get }
    
}

protocol PresentationCoordinator: _PresentationCoordinator {
    
    associatedtype ViewController: UIViewController
    
    var rootViewController: ViewController { get }
    
}

extension PresentationCoordinator {
    
    var _rootViewController: UIViewController {
        return rootViewController
    }
    
}

extension PresentationCoordinator {
    
    func presentCoordinator(_ childCoordinator: _PresentationCoordinator,
                            modalPresentationStyle: UIModalPresentationStyle = .automatic,
                            modalTransitionStyle: UIModalTransitionStyle = .coverVertical,
                            animated: Bool) {
        addChildCoordinator(childCoordinator)
        childCoordinator.start()
        let vc = childCoordinator._rootViewController
        vc.modalPresentationStyle = modalPresentationStyle
        vc.modalTransitionStyle = modalTransitionStyle
        rootViewController.present(childCoordinator._rootViewController, animated: animated)
    }
    
    func dismissCoordinator(_ childCoordinator: _PresentationCoordinator, animated: Bool, completion: (() -> Void)? = nil) {
        childCoordinator._rootViewController.dismiss(animated: animated, completion: completion)
        self.removeChildCoordinator(childCoordinator)
    }

}
// swiftlint:enable all
```

#### NavigationCoordinator.swift

```swift
import UIKit

protocol NavigationCoordinator: PresentationCoordinator {
    
    var navigator: NavigatorType { get }
    
}

extension NavigationCoordinator {
    
    func pushCoordinator(_ childCoordinator: _PresentationCoordinator, animated: Bool) {
        addChildCoordinator(childCoordinator)
        childCoordinator.start()
        navigator.push(childCoordinator._rootViewController, animated: animated) { [weak self] in
            self?.removeChildCoordinator(childCoordinator)
        }
    }
    
}
```

#### Navigator.swift

```swift
import UIKit

protocol NavigatorType {
    
    @discardableResult
    func popToRootViewController(animated: Bool) -> [UIViewController]?

    @discardableResult
    func popToViewController(_ viewController: UIViewController, animated: Bool) -> [UIViewController]?
    
    @discardableResult
    func popViewController(animated: Bool) -> UIViewController?
    
    func push(_ viewController: UIViewController, animated: Bool, onPoppedCompletion: (() -> Void)?)
    
    func setRootViewController(_ viewController: UIViewController, animated: Bool)
}

extension NavigatorType {

    func push(_ viewController: UIViewController, animated: Bool) {
        push(viewController, animated: animated, onPoppedCompletion: nil)
    }

}

class Navigator: NSObject, NavigatorType {

    private let navigationController: UINavigationController
    private var completions: [UIViewController: () -> Void]
    
    init(navigationController: UINavigationController) {
        self.navigationController = navigationController
        self.completions = [:]
        
        super.init()
        
        self.navigationController.delegate = self
    }
    
}

extension Navigator {
    
    func popToRootViewController(animated: Bool) -> [UIViewController]? {
        if let poppedControllers = navigationController.popToRootViewController(animated: animated) {
            poppedControllers.forEach { runCompletion(for: $0) }
            return poppedControllers
        }
        return nil
    }
    
    func popToViewController(_ viewController: UIViewController, animated: Bool) -> [UIViewController]? {
        if let poppedControllers = navigationController.popToViewController(viewController, animated: animated) {
            poppedControllers.forEach { runCompletion(for: $0) }
            return poppedControllers
        }
        return nil
    }
    
    func popViewController(animated: Bool) -> UIViewController? {
        if let poppedController = navigationController.popViewController(animated: animated) {
            runCompletion(for: poppedController)
            return poppedController
        }
        return nil
    }

    func push(_ viewController: UIViewController, animated: Bool, onPoppedCompletion: (() -> Void)? = nil) {
        if let completion = onPoppedCompletion {
            completions[viewController] = completion
        }
        navigationController.pushViewController(viewController, animated: animated)
    }

    func setRootViewController(_ viewController: UIViewController, animated: Bool) {
        completions.forEach { $0.value() }
        completions = [:]
        navigationController.setViewControllers([viewController], animated: animated)
    }
}

extension Navigator: UINavigationControllerDelegate {
    
    func navigationController(_ navigationController: UINavigationController, didShow viewController: UIViewController, animated: Bool) {
        guard let poppingViewController = navigationController.transitionCoordinator?.viewController(forKey: .from),
            !navigationController.viewControllers.contains(poppingViewController) else {
                return
        }
        
        runCompletion(for: poppingViewController)
    }
    
}

private extension Navigator {
    
    func runCompletion(for controller: UIViewController) {
        guard let completion = completions[controller] else { return }
        completion()
        completions.removeValue(forKey: controller)
    }
    
}
```

### ViewController

View 的容器，持有 ViewModel，不包含業務邏輯。具體實作方式（UIKit / SwiftUI）依專案而定。

範例中使用的 `addSubSwiftUIView(_:to:)` 定義如下：

```swift
extension UIViewController {
    func addSubSwiftUIView(_ swiftUIView: some View, to view: UIView) {
        let hostingController = UIHostingController(rootView: swiftUIView)
        hostingController.view.backgroundColor = .clear
        addChild(hostingController)
        view.addSubview(hostingController.view)
        hostingController.view.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            hostingController.view.topAnchor.constraint(equalTo: view.topAnchor),
            hostingController.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            view.bottomAnchor.constraint(equalTo: hostingController.view.bottomAnchor),
            view.trailingAnchor.constraint(equalTo: hostingController.view.trailingAnchor)
        ])
        hostingController.didMove(toParent: self)
    }
}
```

```swift
class ItemListViewController: UIViewController {
    private let viewModel: ItemListViewModel

    init(viewModel: ItemListViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) { fatalError("init(coder:) has not been implemented") }

    override func viewDidLoad() {
        super.viewDidLoad()
        let swiftUIView = ItemListView(viewModel: viewModel)
        addSubSwiftUIView(swiftUIView, to: view)
    }
}
```

### ViewModel

轉發 Interactor 狀態、提供 View 的 action 方法、透過 Coordinator 處理導航。

> 以下範例引用前面段落定義的 `ItemCoordinatorProtocol`（模式 A 範例）與 `ItemListInteractor`（Interactor 範例 1）。

```swift
class ItemListViewModel: ObservableObject {
    @Published private(set) var items: [ItemViewObject] = []
    @Published private(set) var isLoading = false

    private weak var coordinator: ItemCoordinatorProtocol?
    private let interactor: ItemListInteractor
    private var cancellables = Set<AnyCancellable>()

    init(coordinator: ItemCoordinatorProtocol?, interactor: ItemListInteractor) {
        self.coordinator = coordinator
        self.interactor = interactor

        // 在 init 中直接 binding
        interactor.itemsPublisher
            .receive(on: DispatchQueue.main)
            .map { $0.map { ItemViewObject(from: $0) } }
            .sink { [weak self] in self?.items = $0 }
            .store(in: &cancellables)
    }

    func loadItems() {
        Task {
            await MainActor.run { isLoading = true }
            try? await interactor.fetchItems()
            await MainActor.run { isLoading = false }
        }
    }

    func didTapItem(id: String) {
        coordinator?.showDetail(id: id)
    }
}
```

### Interactor

業務邏輯層。介面可以用以下幾種形式暴露，**依實際需求自由組合**：

- **Publisher 屬性**（`var xxxPublisher: Published<T>.Publisher { get }`）：對需要持續推送給 UI 的狀態使用；實作以 `@Published` 持有，這份資料 Interactor 即 source of truth
- **Async throws 方法**（`func xxx(...) async throws -> T`）：對一次性查詢、CRUD action、衍生查詢使用；方法為純 facade，每次呼叫獨立 fetch / mutate
- **Get property**（`var xxx: T { get }`）：對需同步讀當下值的場景使用；通常作為 publisher 狀態的同步讀法

**依資料更新模式選擇形式（per data / per method）：**

| 條件 | Publisher | Async method | Get property |
|------|-----------|--------------|--------------|
| 資料會隨時間變動、UI 要跟著動 | ✅ | ❌ | 搭配 publisher |
| 一次性查詢 / 寫入後即可 | ❌ | ✅ | ❌ |
| 多畫面共享同一份 state | ✅ | ❌ | 搭配 publisher |
| 業務邏輯由多個 UseCase / Repository 組合 | 可以但較重 | ✅ 自然 | ❌ |
| 同步取當下值（不訂閱） | ❌ | ❌ | ✅ |

---

#### 範例 1 — 純 publisher 介面（持續推送狀態）

適用於即時列表更新、播放進度、推播狀態等場景。

```swift
// Protocol — 用 Published<>.Publisher 暴露狀態
protocol ItemListInteractor {
    var itemsPublisher: Published<[ItemDataModel]>.Publisher { get }
    func fetchItems() async throws
}

// 實作 — 用 @Published 持有狀態
class ItemListInteractorImpl: ItemListInteractor {
    @Published private var items: [ItemDataModel] = []
    var itemsPublisher: Published<[ItemDataModel]>.Publisher { $items }

    private let repository: ItemRepository

    init(repository: ItemRepository = .shared) {
        self.repository = repository
    }

    func fetchItems() async throws {
        items = try await repository.fetchItems()
    }
}
```

#### 範例 2 — 純 async method 介面（method facade）

適用於一次性查詢、CRUD action、share link 產生、quota 查詢等場景。protocol 宣告 `async throws` 方法；實作常 compose UseCase / Repository，init 採 default-value DI，測試時注入 mock。

```swift
// Protocol — 純 async 方法 facade
protocol ItemActionInteractor {
    func fetchItem(id: String) async throws -> ItemDTO
    func deleteItem(id: String) async throws
    func updateItem(id: String, title: String) async throws
}

// 實作 — stateless facade，compose UseCase
class ItemActionInteractorImpl: ItemActionInteractor {
    private let fetchUseCase: FetchItemUseCase
    private let deleteUseCase: DeleteItemUseCase
    private let updateUseCase: UpdateItemUseCase

    init(repository: ItemRepository = .shared) {
        self.fetchUseCase = .init(repository: repository)
        self.deleteUseCase = .init(repository: repository)
        self.updateUseCase = .init(repository: repository)
    }

    func fetchItem(id: String) async throws -> ItemDTO {
        try await fetchUseCase.execute(id: id)
    }

    func deleteItem(id: String) async throws {
        try await deleteUseCase.execute(id: id)
    }

    func updateItem(id: String, title: String) async throws {
        try await updateUseCase.execute(id: id, title: title)
    }
}
```

#### 範例 3 — 混用介面（publisher + async method 並存）

實務上同一個 Interactor 經常混用：列表狀態用 publisher 推送即時更新，CRUD action / 衍生查詢用 async method 一次性 return。

```swift
protocol ItemInteractor {
    // Publisher：持續推送的 list 狀態
    var itemsPublisher: Published<[ItemDataModel]>.Publisher { get }
    func fetchItems() async throws

    // Async method：一次性的 CRUD / 衍生查詢
    func deleteItem(id: String) async throws
    func getShareLink(id: String) async throws -> String?
}

class ItemInteractorImpl: ItemInteractor {
    @Published private var items: [ItemDataModel] = []
    var itemsPublisher: Published<[ItemDataModel]>.Publisher { $items }

    private let deleteUseCase: DeleteItemUseCase
    private let shareLinkUseCase: GetShareLinkUseCase

    init(repository: ItemRepository = .shared) {
        self.deleteUseCase = .init(repository: repository)
        self.shareLinkUseCase = .init(repository: repository)
    }

    func fetchItems() async throws {
        items = try await APIEngine.shared.getItems()
    }

    func deleteItem(id: String) async throws {
        try await deleteUseCase.execute(id: id)
        items.removeAll { $0.id == id }  // 同步寫回 publisher state
    }

    func getShareLink(id: String) async throws -> String? {
        try await shareLinkUseCase.execute(id: id)
    }
}
```

**關於 state 同步：** 如果 async method 會影響 publisher 暴露的 state（如上例 `deleteItem` 刪掉 list 中對應 item），在方法 body 內手動同步 `@Published` state，讓 publisher 自動推送更新。

---

## Anti-Patterns

| Bad | Good | 原因 |
|-----|------|------|
| ViewModel 直接呼叫 API | ViewModel 透過 Interactor 取資料 | ViewModel 不應有業務邏輯 |
| ViewController 持有 Interactor | ViewController 只持有 ViewModel | 違反單向依賴 |
| ViewModel 強引用 Coordinator | `weak var coordinator` | 造成循環參考 |
| Interactor protocol 用 `@Published` | 用 `Published<>.Publisher` | Protocol 不支援 property wrapper |
| Coordinator 在 ViewModel 中建立 | Coordinator 建立所有元件 | Coordinator 是唯一的組裝點 |
| 在 ViewModel 中 import 第三方 SDK | 透過 Adapter/Provider 抽象 | 解耦第三方依賴 |
| ViewModel / View 直接呼叫 `push` / `present` | 透過 `coordinator?.showXxx(...)` | 所有導航統一由 Coordinator 處理 |
| Coordinator 把 Interactor 存成 property | Interactor 由各層自行持有，Coordinator 組裝完即放 | Coordinator 只負責組裝 |
| 導航方法接收 Interactor 作為參數 | 接收 data 物件，方法內 init Interactor | 呼叫端不需要知道 Interactor 存在 |
| ViewModel 自己持有 source of truth（自己 fetch / 維護資料） | 由 Interactor 持有，ViewModel 訂閱或讀 `get` property | 資料來源唯一性 |
| 父 Coordinator 用 `presentCoordinator()` 推 child 後不 `dismissCoordinator()` | 對 `presentCoordinator()` 推出的 child 一律 `dismissCoordinator()` 回收 | 避免 `childCoordinators` 殘留 |
| 為一次性 fetch / CRUD 硬塞 `@Published` state + Publisher | 改用 `async throws` 方法直接 return | 無持續更新需求時，publisher 反而增加複雜度 |
| async method 內藏隱性 cache state（每次 call 結果不同） | cache 移到 ViewModel / Repository；method 保持 stateless | 同一個 Interactor 內的 async method 應該是純 facade |

---

## 新功能生成流程

當需要建立新功能畫面時，按以下步驟執行：

### 前置條件

1. **讀取專案 codebase**：找到既有的 Coordinator 基礎設施（`Coordinator.swift` 等四個檔案），確認存在。如果不存在，從上方「Coordinator 基礎設施」段落的程式碼複製到專案中
2. **了解導航結構**：找到 AppCoordinator，了解目前的 Coordinator 樹狀結構
3. **找到參考範本**：找到既有功能的 Coordinator / VC / VM / Interactor 作為風格參考

### 生成步驟

按此順序生成，每一步完成後再進行下一步：

**Step 1 — Interactor（per data / per method 選介面形式）**
- 對每個資料 / 方法分別判斷介面形式：需要持續推給 UI / 多畫面共享 → publisher；一次性 fetch / CRUD / 衍生查詢 → async method；同步取當下值 → get property。多種形式可在同一個 Interactor 並存。
- **Publisher 屬性**：protocol 宣告 `Published<>.Publisher`；實作以 `@Published` 持有狀態並暴露 `var xxxPublisher: Published<Type>.Publisher { $xxx }`
- **Async throws 方法**：protocol 宣告 `async throws`；實作 init 用 default-value DI（如 `init(repository: ItemRepository = .shared)`），方法 body compose UseCase / Repository
- **Get property**：protocol 宣告 `var xxx: T { get }`；實作通常 return `@Published` state 的當下值（如 `var current: Foo { foo }` 對應 `@Published private var foo`），給不需訂閱的場景同步讀
- 若 async method 會改變 publisher state，方法內手動同步 `@Published` state

**Step 2 — ViewModel**
- `ObservableObject`，只持有 UI 狀態（不持有 source of truth）
- `init` 注入 Coordinator protocol（weak）+ Interactor protocol
- 取資料兩種方式：訂閱 Interactor publisher 轉換為顯示資料；或直接讀 Interactor 的 `get` property 同步取當下值
- 提供 View 呼叫的 action 方法（導航一律呼叫 `coordinator?.showXxx(...)`，不直接 push / present）

**Step 3 — ViewController**
- 持有 ViewModel，承載 View
- 具體實作方式（UIKit host SwiftUI / 純 SwiftUI / 純 UIKit）依專案慣例

**Step 4 — Coordinator**
- 定義 protocol（供 ViewModel 弱引用）
- 實作類遵循 `NavigationCoordinator` 或 `PresentationCoordinator`
- 組裝（建立 Interactor → ViewModel → ViewController），組裝位置看是哪種模式：
  - 模式 A（自己建 `UINavigationController`）：組裝放在 `start()`
  - 模式 B（init 接外部 `navigator`）：組裝直接寫在 `init`
- 提供導航方法（push / present 其他 Coordinator），方法接資料物件、方法內 init Interactor
- 若新畫面是既有功能的輕量子畫面（同 navigation stack、不需要獨立流程），可不另建 Coordinator，改成讓父 Coordinator 額外實作此畫面的 `CoordinatorProtocol`（見「多 protocol 實作」段落）

**Step 5 — 接入既有結構**
- 在適當的父 Coordinator 中加入跳轉入口
- 確認新 Coordinator 的 push/present 方式符合導航流程

### 生成原則

- 以當下專案既有程式碼的風格為準，skill 只規範架構分層
- 檔案命名、目錄結構遵循專案既有慣例
- 如果專案有 SwiftLint 等工具，生成的程式碼必須通過檢查
