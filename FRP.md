#FP — Function Programming
##Definition
**Programming Paradigm**
###Feature
1. Function is basic type
2. Expression is a pure calculation process with returned value 

   **y = f(x)** 
   
   🙋🌰
   
   ```
   func rx_getFromRemote(url) -> Observable<Any?> {
       let request = NewsMaster.Network.request(url).get().pack()
       return request.rx_responseJSON()
   }   
   ```
   
3. No "Side Effect"
4. Do not modify gloubal vars
5. Referential transparency

###Resolved
* UT
* Concurrent
* Lazy evaluation

#FRP - Functional Reactive Programming

##Reactive
###Observable
**RACStream RACSignal id**  

**Every operation can be observed**

👓🌰

```
rx_getFromRemote().subscribe(onNext: { _ in
    // do some thing
})
```

##Bind
###Observer
**Subscribe operation is a simple observer**

👓🌰

```
func observer() -> ObserverType {
    return UIBindingObserver()
}
rx_getFromRemote().bindTo(observer())
```

#Practice - MVVM + FRP
##ViewController - Interaction
**Closely related to viewModel, reference ViewModel instance**

* init view & viewModel

```
⚔️
override func viewDidLoad() {
    super.viewDidLoad()
    // init views
    initViews()
    
    // initViewModel
    let viewModel = SubscriptionExploreViewModel<SubscriptionExploreItem>(input: (language: languageChange.asDriver(),
                                                                                  pageReset: resetVariable.asDriver(),
                                                                                  retryLoading: retryLoading.asDriver()))
    // bindViewModel
    bindDataSource(viewModel)
    
    // start state changes, then trigger first request 
    self.reset()
}

🛡
/** 
 languageChange & resetVariable & retryLoading maybe triggered change by user interaction 
 then the change will trigger viewModel update data 
 and then new data will update views by existed binding relationships
 */

```

* bind view & viewModel

```
⚔️
priv func bindDataSource(_ viewModel: SubscriptionExploreViewModel<SubscriptionExploreItem>) {
     dataSource.configureCell = { [weak self] (dataSource, tableView, indexPath, subscriptionExploreItem) -> UITableViewCell in
         var cell: UITableViewCell
         if indexPath.row < dataSource[0].items.count {
             let items = dataSource[0].items
             let exploreItem = items[indexPath.row]
             if let sources = exploreItem.sources, let sourceCell = tableView.dequeueReusableCell(withIdentifier: "SubscriptionSourcesGridIdentifier", for: indexPath) as? SubscriptionSourcesGridCell {
                 sourceCell.sources = sources
                 sourceCell.delegate = self
                 sourceCell.sourceParameters = self?.eventViewAsSourceParameters()
                 cell = sourceCell
             } else {
                 let exploreCell = tableView.dequeueReusableCell(withIdentifier: "SubscribeSourceIndicatorIdentifier", for: indexPath) as! SubscribeSourceIndicatorCell
                 exploreCell.exploreItem = exploreItem
                 cell = exploreCell
             }
         } else {
             cell = tableView.dequeueReusableCell(withIdentifier: "default")!
         }
         return cell
     }
     
     // bind tableView
     let observable = viewModel.itemsVariable.asObservable()
         .observeOn(MainScheduler.instance);
     observable.map({ (items) -> [SectionModel<String, SubscriptionExploreItem>] in
         return [SectionModel(model: "section", items: items)]
     })
         .bindTo(self.tableView!.rx.items(dataSource: dataSource))
         .disposed(by: disposeBag)
     
     // bind pageView
     viewModel.pageStateVariable.asDriver().drive(self.pageLoadingState())
         .disposed(by: disposeBag)
     
     // bind tabView`s events
     self.tableView!.rx.willDisplayCell.map({ (cell, indexPath) -> UITableViewCell in
         return cell
     }).bindTo(self.willDisplayCell()).disposed(by: disposeBag)
     
     self.tableView!.rx.itemSelected.bindTo(self.itemSelected()).disposed(by: disposeBag)
     
     // others
     viewModel.subscribeContentChange.bindTo(self.subscribeContentChange()).disposed(by: disposeBag)
     
     self.rx.sentMessage(#selector(subscriptionSource(_:didClickedFollowButton:))).asObservable().bindTo(self.subscriptionSourceObserver()).disposed(by: disposeBag)
}
```

##ViewModel - DataSource
**Closely related to ViewController, but do not dependent directly**

```
⚔️
// output
var pageStateVariable = Variable(PageState.normal)
var subscriptionsCountVariable:Driver<Int>
var subscribeContentChange:Observable<Notification>

init(input:(language:Driver<Bool>,
           pageReset:Driver<Bool>,
        retryLoading:Driver<Bool>)) {
    subscriptionsCountVariable = SubscribeCenter.default.sourcesCountVariable.asDriver()
    subscribeContentChange = NotificationCenter.default.rx.notification(Notification.Name(rawValue: SubscriptionInfoDidChangeNotification), object: nil).observeOn(MainScheduler.instance)
    super.init()
    
    Driver.combineLatest(input.language, input.pageReset, input.retryLoading) {($0, $1, $2)}.throttle(0.5).drive(onNext: { [weak self] (value1, value2, value3) in
        guard let strongSelf = self, value1 || value2 || value3 else { return }
        strongSelf.fetchRemoteData()
    }).disposed(by: disposeBag)
}

```

##Net | DB - Request
**One Side**

ViewModel -> fetch & bind -> Result

```

📦
extension AFRequest {
    func rx_responseJSON() -> Observable<[String : Any]?> {
        return Observable.create({ [weak self] (observer) -> Disposable in
            var disposables:Cancelable?
            disposables = Disposables.create {
                if !(disposables!.isDisposed) {
                    self?.cancel()
                }
            }
            self?.nm_responseJSON(){ (request, response, any, error) in
                if let onError = error {
                    observer.onError(onError)
                } else {
                    observer.onNext(any)
                    observer.onCompleted()
                }
            }
            return disposables!
        }).shareReplay(1)
    }
}

📦
extension ViewModel {
    func rx_getFromRemote(_ source: LoadSource = .refresh(false)) -> Observable<[String : Any]?> {
        guard let request = NewsMaster.Network.request(requestURLWithLoadSource(source)).get(requestParametersWithLoadSource(source)).pack() else {
            return Observable.create({ (observer) -> Disposable in
                observer.onError(NewsMaster.Error(userInfo: ["message" : "not a valid url"]))
                return Disposables.create()
            })
        }
        return request.rx_responseJSON()
    }
}

🍐
func fetchRemoteData() -> Observable<[SubscriptionExploreItem]> {
    return rx_getFromRemote().map({ (results) -> [SubscriptionExploreItem] in
        return self.parseResponse(results: results)!
    })
}

🌰
fileprivate func fetchRemoteData() {
    rx_getFromRemote().map({ results -> [SubscriptionExploreItem] in
        return self.parseResponse(results: results)!
    }).catchError({ [weak self] (error) -> Observable<[SubscriptionExploreItem]> in
        guard let weakSelf = self, let customError = error as? NewsMaster.Error else {return Observable.empty()}
        
        weakSelf.pageStateVariable.value = PageState.error(PageStateImageType.connectionFailed.imageName(), customError.errorMessage, true)
        return Observable.empty()
    }).subscribe(onNext: { [weak self] (items) in  // side effect
        guard let weakSelf = self else {return}
        
        weakSelf.pageStateVariable.value = PageState.normal
        weakSelf.items = items
        
        let subscriptions = items.reduce([]) {
            (result: [SubscriptionItem], e: SubscriptionExploreItem) -> [SubscriptionItem] in
            guard let sources = e.sources else {
                return result
            }
            var res = result
            res.append(contentsOf: sources)
            return res
        }
        SubscribeCenter.default.synchronizeSubscriptions(subscriptions)
    }).disposed(by: disposeBag)
}

```
