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

- **Interactor** 以 `@Published` 持有業務狀態，protocol 介面透過 `Published<>.Publisher` 暴露（因為 protocol 不支援 `@Published`）
- **ViewModel** 持有 `@Published` 顯示狀態，訂閱 Interactor 的 Publisher 並轉換為 View 所需資料
- **ViewController** 是容器，持有 ViewModel 供 View 觀察

### 3. 弱引用規則

- Coordinator 中 `weak var parentCoordinator`
- ViewModel 中 `weak var coordinator`（protocol 型別）
- 所有可能造成循環參考的地方用 `weak`

### 4. Protocol 注入

- Coordinator protocol、Interactor protocol 皆以 protocol 型別傳入
- 具體型別只在 Coordinator 組裝時出現
- 便於測試與解耦

**命名慣例：**
- Coordinator protocol：`{功能名}CoordinatorProtocol`（如 `HomeCoordinatorProtocol`、`TabCoordinatorProtocol`）
- Interactor protocol：`{功能名}InteractorProtocol`（如 `HomeInteractorProtocol`）
- Interactor 實作：`{功能名}InteractorImpl`（如 `HomeInteractorImpl`）

### 5. Publisher 轉發鏈

```
Interactor(@Published) → ViewModel(subscribe + transform) → View(@ObservedObject)
```

- ViewModel 在 `init` 中直接訂閱 Interactor 的 Publisher
- 使用 `.receive(on: DispatchQueue.main)` 確保 UI 執行緒

### 6. 檔案內容排列順序

每個 Swift 檔案內的成員按以下順序排列：

1. **變數**（properties） — `@Published`、`weak var`、`private var`、`let` 等
2. **`init`** — 初始化方法
3. **func** — 方法（public/internal 在前，private 在後）

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
- 透過基礎設施的 `pushCoordinator()` / `presentCoordinator()` 管理畫面跳轉，child 生命週期由基礎設施自動管理

```swift
protocol ItemCoordinatorProtocol: AnyObject {
    func showDetail(id: String)
}

class ItemCoordinator: NavigationCoordinator, ItemCoordinatorProtocol {
    var rootViewController: UINavigationController
    var navigator: NavigatorType
    weak var parentCoordinator: Coordinator?
    var childCoordinators: [Coordinator] = []

    init(navigator: NavigatorType) {
        self.navigator = navigator
        self.rootViewController = UINavigationController()
    }

    func start() {
        let interactor = ItemListInteractorImpl()
        let viewModel = ItemListViewModel(coordinator: self, interactor: interactor)
        let viewController = ItemListViewController(viewModel: viewModel)
        rootViewController.setViewControllers([viewController], animated: false)
    }

    func showDetail(id: String) {
        let detailCoordinator = ItemDetailCoordinator(itemID: id, navigator: navigator)
        pushCoordinator(detailCoordinator, animated: true)
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
class DetailCoordinator: NavigationCoordinator {
    weak var parentCoordinator: Coordinator?
    var childCoordinators: [Coordinator] = []
    let rootViewController: UIViewController
    let navigator: NavigatorType

    init(interactor: DetailInteractorProtocol, navigator: NavigatorType) {
        self.navigator = navigator
        let viewModel = DetailViewModel(interactor: interactor)
        let vc = DetailViewController(viewModel: viewModel)
        vc.hidesBottomBarWhenPushed = true
        self.rootViewController = vc
    }

    func start() {}
}

// 父 Coordinator 中使用
func showDetail(interactor: DetailInteractorProtocol) {
    let coordinator = DetailCoordinator(interactor: interactor, navigator: navigator)
    pushCoordinator(coordinator, animated: true)
}
```

**何時自建 vs 共用 Navigator：**
- **自建**：Tab 的根 Coordinator、獨立流程（如 Onboarding）— 需要自己的 `UINavigationController`
- **共用**：同一個 navigation stack 內的子頁面 — 接收父 Coordinator 的 `navigator`，`init` 中組裝完畢，`start()` 為空

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

業務邏輯層，擁有狀態，透過 protocol 暴露 Publisher。

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

    func fetchItems() async throws {
        let result = try await APIEngine.shared.getItems()
        items = result
    }
}
```

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

---

## 新功能生成流程

當需要建立新功能畫面時，按以下步驟執行：

### 前置條件

1. **讀取專案 codebase**：找到既有的 Coordinator 基礎設施（`Coordinator.swift` 等四個檔案），確認存在。如果不存在，從上方「Coordinator 基礎設施」段落的程式碼複製到專案中
2. **了解導航結構**：找到 AppCoordinator，了解目前的 Coordinator 樹狀結構
3. **找到參考範本**：找到既有功能的 Coordinator / VC / VM / Interactor 作為風格參考

### 生成步驟

按此順序生成，每一步完成後再進行下一步：

**Step 1 — Interactor**
- 定義 protocol，宣告 `Published<>.Publisher` 屬性與方法
- 建立實作類，以 `@Published` 持有業務狀態
- 暴露 publisher：`var xxxPublisher: Published<Type>.Publisher { $xxx }`

**Step 2 — ViewModel**
- `ObservableObject`，持有 `@Published` 顯示狀態
- `init` 注入 Coordinator protocol（weak）+ Interactor protocol
- `init` 中訂閱 Interactor publisher，轉換為顯示資料
- 提供 View 呼叫的 action 方法（導航交給 coordinator）

**Step 3 — ViewController**
- 持有 ViewModel，承載 View
- 具體實作方式（UIKit host SwiftUI / 純 SwiftUI / 純 UIKit）依專案慣例

**Step 4 — Coordinator**
- 定義 protocol（供 ViewModel 弱引用）
- 實作類遵循 `NavigationCoordinator` 或 `PresentationCoordinator`
- `start()` 中組裝：建立 Interactor → ViewModel → ViewController
- 提供導航方法（push/present 其他 Coordinator）

**Step 5 — 接入既有結構**
- 在適當的父 Coordinator 中加入跳轉入口
- 確認新 Coordinator 的 push/present 方式符合導航流程

### 生成原則

- 以當下專案既有程式碼的風格為準，skill 只規範架構分層
- 檔案命名、目錄結構遵循專案既有慣例
- 如果專案有 SwiftLint 等工具，生成的程式碼必須通過檢查
