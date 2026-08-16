---
layout: post
title:  "Rethinking view state management in SwiftUI without ViewModel"
date:   2026-08-16 16:00:00 +0200
categories: swiftui viewmodel dependency-injection architecture
---

Around the web, I've seen quite a few examples that promote the use of a ViewModel to manage the view state of a SwiftUI `View`.

Applying this pattern means having, alongside a view — e.g. `MyView` — its counterpart ViewModel, e.g. `MyViewModel`.

`MyViewModel` could look like the following, where the `State` enumeration is exposed to the outside world through a `state` property.

```swift
@Observable
final class MyViewModel {
    enum State {
        case idle
        case loading
        case loaded([SampleData])
        case error(String)
    }
    
    let service: Service = LiveService()
    
    private(set) var state: State = .idle
    // store `sampleData` if you need to retrieve / listen to changes from other views
    private(set) var sampleData: [SampleData] = []
    
    func fetchData() async {
        state = .loading
        
        do {
            let data = try await service.fetchData()
            sampleData = data
            state = .loaded(data)
        } catch {
            state = .error(error.localizedDescription)
        }
    }
}
```

Since `MyViewModel` is registered as `Observable`, `MyView` observes changes to its properties and can react accordingly. This is achieved by declaring `MyViewModel` as a `@State` property.

```swift
struct MyView: View {
    @State private var viewModel = MyViewModel()
    
    var body: some View {
        ZStack {
            switch viewModel.state {
            case .idle, .loading:
                ProgressView()
            case .loaded(let sampleData):
                List(sampleData) { item in
                    Text(item.name)
                }
            case .error(let message):
                Text("Error: \(message)")
            }
        }
        .task {
            await viewModel.fetchData()
        }
    }
}

#Preview {
    MyView()
}
```

Note that, as written, there's no way to inject anything different here: `viewModel` is created as a plain property default with no parameters, so the preview above is stuck with whatever `MyViewModel()`'s default init produces — the real `LiveService`. Fixing that starts by giving `MyViewModel` an injectable service:

```swift
@Observable
final class MyViewModel {
    // ...
    let service: Service

    init(service: Service = LiveService()) {
        self.service = service
    }
    // ...
}
```

It might then seem intuitive to initialize `MyViewModel` from within the view's `init` to satisfy a dependency passed in from a parent, but this can lead to stale data that fails to update.

```swift
struct MyView: View {
    @State private var viewModel: MyViewModel

    init(service: Service) {
        _viewModel = State(wrappedValue: MyViewModel(service: service))
    }
    
    // ...
}
```

Apple's documentation is clear on this topic [1]:

> You don't call this initializer directly. Instead, SwiftUI calls it for you when you declare a property with the `@State` attribute and provide an initial value.

Also worth noting, `State` provides another initializer called `init(initialValue:)`, which behaves the same as the one above. `init(initialValue:)` was the original name in the first Property Wrappers proposal; it was later renamed to `init(wrappedValue:)`, but the original was kept for backward compatibility [2].

There are, of course, other ways to work around this — for instance, using a Dependency Injection framework such as Factory [3] or swift-dependencies [4]. But before reaching for an external framework, is there a simpler alternative?

Well, I've been thinking about it, and while working on a new SwiftUI application, another solution came to mind. Maybe someone has already tried this and I'm just not aware of it — hence this post.

Instead of using a ViewModel to manage a view's state, why can't the view manage its own screen state directly?

```swift
struct MyView: View {
    enum ViewScreen {
        case initial
        case loading
        case main([SampleData])
        case error(String)
    }
    
    @State private var viewScreen = ViewScreen.initial
    
    let service: Service
    
    init(service: Service = LiveService()) {
        self.service = service
    }
    
    var body: some View {
        ZStack {
            switch viewScreen {
            case .initial:
                EmptyView()
            case .loading:
                ProgressView()
            case .main(let sampleData):
                List(sampleData) { item in
                    Text(item.name)
                }
            case .error(let message):
                Text("Error: \(message)")
            }
        }
        .task {
            viewScreen = .loading
            do {
                let sampleData = try await service.fetchData()
                viewScreen = .main(sampleData)
            } catch {
                viewScreen = .error(error.localizedDescription)
            }
        }
    }
}

#Preview {
    MyView(service: TestService())
}
```

As you can see from the example above, `MyView` is now in charge of managing its own screen state, while the service has become a plain, injectable dependency — no need for a workaround `init`, and the preview can pass a `TestService` directly.

There's one more step to unlock the full potential of this approach. Consider another view in the application that needs to react to changes happening during `MyView`'s task execution — for instance, showing a badge or a summary based on the data `MyView` just fetched. For that, I can introduce a separate `Observable` class that holds the piece of application state, or business logic, other views need to read. Using `@Environment`, I can share that class across views without relying on an external Dependency Injection mechanism.

```swift
@Observable
final class MyModel {    
    private(set) var myInterestingData: SampleData?
    
    func updateInterestingData(with sampleData: [SampleData]) {
        // ...
        myInterestingData = sampleData.first
    }
}

struct MyView: View {
    enum ViewScreen {
        case initial
        case loading
        case main([SampleData])
        case error(String)
    }
    
    @State private var viewScreen = ViewScreen.initial
    @Environment(MyModel.self) private var myModel
    
    let service: Service
    
    init(service: Service = LiveService()) {
        self.service = service
    }
    
    var body: some View {
        ZStack {
            switch viewScreen {
            case .initial:
                EmptyView()
            case .loading:
                ProgressView()
            case .main(let sampleData):
                List(sampleData) { item in
                    Text(item.name)
                }
            case .error(let message):
                Text("Error: \(message)")
            }
        }
        .task {
            viewScreen = .loading
            do {
                let sampleData = try await service.fetchData()
                myModel.updateInterestingData(with: sampleData)
                viewScreen = .main(sampleData)
            } catch {
                viewScreen = .error(error.localizedDescription)
            }
        }
    }
}

#Preview {
    MyView(service: TestService())
        .environment(MyModel())
}
```

A couple of things worth calling out in this last example. `@Environment(MyModel.self)` only works if something above `MyView` in the hierarchy — or the `#Preview` itself, as shown here — has actually placed a `MyModel` instance into the environment via `.environment(MyModel())`; without it, the app crashes at runtime the moment `myModel` is accessed. Also notice that two different kinds of state now live side by side: `viewScreen` stays local to `MyView` and drives what it renders, while `myModel` is shared state other views can also read. Inside `.task`, updating `myModel` and updating `viewScreen` are two separate, independent state changes — not two steps of the same operation.

With this approach, `MyView` becomes the entry point for its own screen: dependencies like `service` are injected as plain, ordinary properties — not hidden inside a ViewModel wrapped in `@State` — and any state that needs to be shared with other views is handled through `@Environment` and `@Observable`, without reaching for an external Dependency Injection framework.

The trade-off is testability — though a narrower one than it might first appear, and one you're not stuck with. The `Service` itself is a plain, protocol-based type, and stays exactly as testable in isolation as it was before, regardless of which pattern owns it. What moves into the view's `.task { }` closure is the thin orchestration layer that used to live in `MyViewModel.fetchData()`: mapping the service's success or failure into `.main`/`.error`. Left inline, that glue code is no longer exercised by a plain XCTest — testing it means testing the view itself.

But it doesn't have to stay inline. If the orchestration grows non-trivial — retries, combining multiple calls, non-trivial mapping — you can factor it out into its own type and inject it exactly like `service`, as a plain `let` property. The only rule is that this type must not be `@Observable`. As long as it stays a plain object, it's fully unit-testable on its own, and — since it's never wrapped in `@State` — none of the stale-data issues from earlier come back either.

In short: this isn't a rejection of dedicated, testable types for business logic — it's a rejection of making that type `@Observable` and holding it in `@State`. That combination is what tied a ViewModel to the view's lifecycle and caused the injection problems in the first place. Keep the orchestration in a plain object, inject it like any other dependency, and you get testability and simple dependency injection at the same time.

#### References

[1] https://developer.apple.com/documentation/swiftui/state/init(wrappedvalue:)

[2] https://forums.swift.org/t/swiftui-state-init-wrappedvalue-vs-init-initialvalue-whats-the-difference/38904/2

[3] https://github.com/hmlongco/Factory

[4] https://github.com/pointfreeco/swift-dependencies
