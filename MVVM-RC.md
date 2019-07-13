# MVVM-RC [WIP]

## Wha?

An opinionated iOS ~~architectural~~ app building pattern which embraces rapid app development. The pain point that it tries to address is the code complexity and clutter of adding/removing screens(VCs) to/from application flow.

## Inspired by
This is inspired by ASP.NET MVC and similar web app frameworks for screen building and navigation and canonical MVVM for the "front-end". 

## Prerequisites

Although the architecture can be implemented w/o these, in the spirit of rapid development it assumes that the app is integrating a DI container solution (Swinject) and a reactive UI solution (RxCocoa).

## General pros and cons compared to other popular architectures 

### Pros
* Well defined/templated way of adding screens with minimal code.
* Genericized and reusable navigation code.
* Deep linking is very easy to implement.

### Cons
* VMs carry the responsibility of knowing where, how and when to route the app to.
* In the app lifecycle the screens are throw-away from memory management POV and caching may need to be added to this architecture.
* Kind of forces your to genericize all possible VC transitions including those involved in partial views (containerized VCs).

### ProCon
* Discourages passing data between routes - assumes state persistence by a service or a model. This is still possible and useful for cases when data being passed between routes is purely for navigational purpose.

## Pieces
* MVVM - Model–view–viewmodel - this works the same way as in every other architecture. Simple solution: bind your VC to VM using RxCocoa.
* R - Router - responsible for all types of transitions between routes. Facility for transitions has to be generacized, so router shouldn't know about peculiarities of a particular route. Also responsible for knowing where the app currently is and how it got there. Bonus: can easily route the app using deep links and trace the navigation history. Router is a services injected into VMs. VMs use the Router navigate Routes.
* C - Coordinator - responsible for routing the app to entry screen, responding to routing requests from services(ie. user got logged out take to sign in page) and restoring application UI state. Coordinator uses services to infer the UI state and uses Router to set the state. Most likely your app needs only 1 coordinator and it will probably be very lean even if your app is complex.
* Route - Essentially a VC factory. Implementation can vary, refer to examples for basic implementation pattern.

### Discussion and FAQ

* Assembly introspection and convention based Route locator can eliminate the need of registering Routes with the Router
* What if my Route supports only some types of transitions? - up to you how to handle violations, most likely you'd want to handle these in your implementation of the Router. Since this can only be determined at runtime, recommendation is to assert failure and to reveal the defects through UI tests or integration tests.
* What if my Route's dependencies have a potential of not being satisfied in some theoretical routes or What if someone tries to route to my route without setting up my route's dependencies correctly - up to you how to handle. The recommendation is to handle these violations in VMs. Violations of injected objects/values is likely to cause either compile time issues or runtime crashes will can be caught through UI and integration tests. For optional property dependencies and flow specific data dependencies it is TBD on what's the best approach to handle. So far it seems like handling gracefully through some sort of UI message is most practical and development screw ups are to be handled through UI and integration tests.
* Convention based storyboard loading helps to cut down on boilerplate VC instantiation: 
```
extension UIViewController {
    static func instantiate<T>() -> T where T: UIViewController {
        let mirror = Mirror(reflecting: self)
        let typeName = "\(mirror.subjectType)"
        let storyboardName = typeName.replacingOccurrences(of: "ViewController.Type", with: "")
        return UIStoryboard(name: storyboardName, bundle: nil).instantiateInitialViewController()! as! T
    }
}
```
... beware of getting your VC and storyboard names out of sync.
* Testability? - unit tests are as straightforward as it gets. Integration tests, UI tests - arguably easier than in many other architectures since Router gives easy access to setting up app states and spying on router allows to capture the transitions.
* Does this actually help to save dev time? - judge yourself. It is pretty DRY.
* The app flow logic hides in VMs, how does one read/infer the actual app flow? - false... coordinator will tell the big picture and the rest of the flow would have to be inferred from VMs, however you just need to search "router" in any given VM to find all possible transitions from given VM/route.

## Examples (soz sketch code)
### Basic routing
```
class HomeVM {
	...
	func addSubscriptions (
		didTapSettings.subscribe {
			router.route(SettingsRoute.self, navigationType: .push)
		}
		didTapHelp.subscribe {
			router.route(HelpRoute.self, navigationType: .modal)
		}
		didTapSignOut {
			router.route(AuthenticationRoute.self, navigationType: .crossFade) {
				// AuthenticationRoute here as an example has a closure dependency it runs when // it completes, so in this case it requires this closure so that caller 
				// instructs it what to do once the auhtentication is done 
				router.route(HomeRoute.self, navigationType: .crossFade)
			}
		}
	) 
	...
}

class SettingsRoute: Route {
	let vc: UIViewController
	init(rootContainer: Container) {
		let vc = SettingsViewController.isntantiate()
		let settingsContainer = rootContainer.derive()
        settingsContainer.autoregister(SettingsVM.self, initializer: SettingsVM.init)
		let vm = container.resolve(SettingsVM.self)
		vc.vm = vm
		self.vc = vc
	}
}

class AuthenticationRoute: Route {
	let vc: UIViewController
	// Cheating here, in practice this would probably be a convenience init which wraps the container and calls the required init
	init(rootContainer: Container, authFlowCompletion: @escaping () -> Void) {
		let vc = AuthenticationViewController.isntantiate()
		let authenticationContainer = rootContainer.derive()
        authenticationContainer.autoregister(AuthenticationVM.self, initializer: AuthenticationVM.init, arguments: @escaping () -> Void)
		let vm = container.resolve(AuthenticationVM.self, arguments: authFlowCompletion)
		vc.vm = vm
		self.vc = vc
	}
}
```
### Coordinator
```
class AppFlowCoordinator {
	...
	func appStart() {
		if userManager.isUserOnboarded {
			router.route(HomeRoute.self, navigationType: .crossFade)
		} else {
			router.route(WelcomeRoute.self, navigationType: .crossFade)
		}
	}
	...
	func observeEvents() {
		userManager.didSignOut.subscribe {
			router.route(SignInRoute.self, navigationType: .modal)
		}
		serverNotificationManager.didReceiveSuperUrgentNotificaiton { notification in
			router.dismissAllModals()
			router.route(NotificationRoute.self, navigationType: .modal, arguments: notification)
		}	
	}
}
```

