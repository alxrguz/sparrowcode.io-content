Сегодня научимся изменять порядок ячеек, перетаскивать ячейки группами, перемещать ячейки между коллекциями и даже между приложениями. Разберём для `UICollectionView` и `UITableView`.

Перед погружением в код разберёмся, как устроен жизненный цикл драга и дропа.

![Кадр из фильма «Форсаж: Хоббс и Шоу».](https://cdn.sparrowcode.io/tutorials/drag-and-drop/preview.jpg)

## Модели

Драг отвечает за перемещение объекта, а дроп — за сброс объекта и новое положение. Когда палец с ячейкой ползёт по экрану, вызывается метод делегата. Очень похоже на `UIScrollViewDelegate` с методом `scrollViewDidScroll`.

`UIDragSession` и `UIDropSession` это объекты-обёртки с информацией о положении пальца, объектов, для которых совершали действия, кастомного context и т.д. Перед началом драга предоставьте объект `UIDragItem`. Это должен быть не класс ячейки. Передавайте объект, который представляет данные - например модель пиццы если у вас коллекция с пиццами.

```swift
let itemProvider = NSItemProvider.init(object: yourObject)
let dragItem = UIDragItem(itemProvider: itemProvider)
dragItem.localObject = action
return dragItem
```

Чтобы провайдер смог скушать любой объект, реализуйте протокол `NSItemProviderWriting`:

```swift
extension YourClass: NSItemProviderWriting {
    
    public static var writableTypeIdentifiersForItemProvider: [String] {
        return ["YourClass"]
    }
    
    public func loadData(withTypeIdentifier typeIdentifier: String, forItemProviderCompletionHandler completionHandler: @escaping (Data?, Error?) -> Void) -> Progress? {
        return nil
    }
}
```

Мы готовы. Потянули!

## Drag

### Одна ячейка

Разберем на примере коллекции. Советую использовать `UICollectionViewController`, из коробки он умеет больше. Но и простая collection-вью подойдёт.

Установим драг-делегат:

```swift
class CollectionController: UICollectionViewController {
    
    func viewDidLoad() {
        super.viewDidLoad()
        collectionView.dragDelegate = self
    }
}
```

Реализуем протокол `UICollectionViewDragDelegate`. Первым будет метод `itemsForBeginning`:

```swift
func collectionView(_ collectionView: UICollectionView, itemsForBeginning session: UIDragSession, at indexPath: IndexPath) -> [UIDragItem] {
    let itemProvider = NSItemProvider.init(object: yourObject)
    let dragItem = UIDragItem(itemProvider: itemProvider)
    dragItem.localObject = action
    return dragItem
}
```

Вы уже видели этот код выше. Он оборачивает наш объект в `UIDragItem`. Метод вызывается при подозрении, что пользователь хочет начать драг. 

> Не используйте этот метод как начало драга, потому что его вызов только предполагает, что драг начнётся.

Добавим ещё два метода — `dragSessionWillBegin` и `dragSessionDidEnd`:

```swift
extension CollectionController: UICollectionViewDragDelegate {
    
    func collectionView(_ collectionView: UICollectionView, itemsForBeginning session: UIDragSession, at indexPath: IndexPath) -> [UIDragItem] {
        let itemProvider = NSItemProvider.init(object: yourObject)
        let dragItem = UIDragItem(itemProvider: itemProvider)
        dragItem.localObject = action
        return dragItem
    }
    
    func collectionView(_ collectionView: UICollectionView, dragSessionWillBegin session: UIDragSession) {
        
    }
    
    func collectionView(_ collectionView: UICollectionView, dragSessionDidEnd session: UIDragSession) {
        
    }
}
```

Первый метод вызывается, когда драг начался, а второй - когда драг закончился. Перед `dragSessionWillBegin` вызывается `itemsForBeginning`. Но не факт, что если вызвался `itemsForBeginning`, вызовется метод `dragSessionWillBegin`. Если хотите обновить интерфейс на время драга, например, спрятать кнопки удаления, `dragSessionWillBegin` правильное место. 

Давайте посмотрим, что получается на этом этапе.

[Начало и завершение работы драга.](https://cdn.sparrowcode.io/tutorials/drag-and-drop/drag-delegate.mov)

Ячейка возвращается на место потому что дроп еще не готов, его реализуем дальше.

### Несколько ячеек

В протоколе `UICollectionViewDragDelegate` мы реализовывали метод `itemsForBeginning`, который возвращал объект драга. Чтобы к текущему драгу добавить ещё объекты, реализуйте метод `itemsForAddingTo`:

```swift
func collectionView(_ collectionView: UICollectionView, itemsForAddingTo session: UIDragSession, at indexPath: IndexPath, point: CGPoint) -> [UIDragItem] {
    // Код аналогичен. Создаём `UIDragItem` на основе нашего объекта:
    let itemProvider = NSItemProvider.init(object: yourObject)
    let dragItem = UIDragItem(itemProvider: itemProvider)
    dragItem.localObject = action
    return dragItem
}
```

Теперь ячейки собираются в стопку. Стопку можно сбрасывать как отдельные ячейки.

[Сбор ячеек в стопку во время драга.](https://cdn.sparrowcode.io/tutorials/drag-and-drop/drag-stack.mov)

## Drop

Драг - половина дела. Теперь научимся сбрасывать ячейку.

### Для `CollectionView`

Реализуем протокол `UICollectionViewDropDelegate`:

```swift
extension CollectionController: UICollectionViewDropDelegate {
    
    func collectionView(_ collectionView: UICollectionView, dropSessionDidUpdate session: UIDropSession, withDestinationIndexPath destinationIndexPath: IndexPath?) -> UICollectionViewDropProposal {
        
    }
    
    func collectionView(_ collectionView: UICollectionView, performDropWith coordinator: UICollectionViewDropCoordinator) {
        
    }
    
    func collectionView(_ collectionView: UICollectionView, dropSessionDidEnd session: UIDropSession) {
        
    }
}
```

Первый метод требует вернуть объект `UICollectionViewDropProposal`. Метод отвечает за превью и обновление интерфейса, подсказывает пользователю, что произойдёт, если дроп сделать сейчас.

Вернуть можно один из нескольких статусов, разберём каждый:

```swift
// Ячейка вернётся на место, визуальные индикаторы не появятся. Действие не смещает другие ячейки.
return .init(operation: .cancel)

// Появится серая перечеркнутая иконка. Это значит, что операция запрещена.
return .init(operation: .forbidden)

// Произойдёт полезное действие, визуальные индикаторы не появятся.
return .init(operation: .move)

// Ячейки смещаются для предлагаемого места дропа, визуальные индикаторы не появятся.
return .init(operation: .move, intent: .insertAtDestinationIndexPath)

// Появляется зелёный плюс — индикатор копирования.
return .init(operation: .copy)
```

В нашем примере сделаем так - если есть прогнозируемый `IndexPath`, то разрешаем сброс. Если нет - то запрещаем. Лучше поставить отмену, но так будет нагляднее.

```swift
func collectionView(_ collectionView: UICollectionView, dropSessionDidUpdate session: UIDropSession, withDestinationIndexPath destinationIndexPath: IndexPath?) -> UICollectionViewDropProposal {
        
    guard let _ = destinationIndexPath else { return .init(operation: .forbidden) }
    return .init(operation: .move, intent: .insertAtDestinationIndexPath)
}
```

`destinationIndexPath` — системный расчёт, куда ячейку можно дропнуть. Он ни к чему не обязывает, более того, дропнуть мы можем в другое место. 

Теперь перейдём к следующему методу `performDropWith`. Здесь решаем самые главные дела: меняем данные, переставляем ячейки и уведомляем систему, куда дропнули вьюху, чтобы система отрисовала анимацию.

```swift
func collectionView(_ collectionView: UICollectionView, performDropWith coordinator: UICollectionViewDropCoordinator) {
        
    // Если система не смогла определить IndexPath, то останавливаем выполнение.
    // Дальше мы научимся определять индекс самостоятельно, но пока оставим так.
    guard let destinationIndexPath = coordinator.destinationIndexPath else { return }
        
    for item in coordinator.items {
        // Получаем доступ к нашему объекту, приводим тип.
        guard let yourObject = item.dragItem.localObject as? YourClass else { continue }
        // Объект перемещаем из одного места в другое. Я использую псевдофункцию, подразумеваю кастомную логику:
        move(object: yourObject, to: destinationIndexPath)
    }
        
    // Не забудьте обновить коллекцию.
    // Если используете классический data source, изменения вносите в блоке `performBatchUpdates`.
    // Если у вас diffable data source, используйте обновление снепшота.
    // Функция для примера, такой функции нет.
    collectionView.reloadAnimatable()
        
    // Уведомляем, куда сбросили элемент.
    // Самостоятельно реализуйте функцию `getIndexPath`.
    for item in coordinator.items {
        guard let yourObject = item.dragItem.localObject as? YourClass else { continue }
        if let indexPath = getIndexPath(for: yourObject) {
            coordinator.drop(item.dragItem, toItemAt: indexPath)
        }
    }
}
```

Теперь коллекция и data source обновляются при перемещении, ячейка дропается по новому индексу. Глянем, что получилось:

[Перемещение и дроп ячейки в коллекцию.](https://cdn.sparrowcode.io/tutorials/drag-and-drop/drop-delegate.mov)

Чтобы ячейки расступались для дропа другой ячейки, используйте Drop Proposal c `.insertAtDestinationIndexPath`. Любой другой интент не будет этого делать. Иногда багует с коллекцией, будьте осторожны.

При попытке сбросить ячейку последней `FlowLayout` запросит несуществующие атрибуты ячейки. Когда ячейки расступаются, лейаут рисует ячейку внутри, а при дропе получается ячеек больше, чем моделей в Data Source. Это решается переопределением метода в `UICollectionViewFlowLayout`:

```swift
override func layoutAttributesForItem(at indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
    if let countItems = collectionView?.numberOfItems(inSection: indexPath.section) {
        if countItems == indexPath.row {
            // If ask layout cell which not isset,
            // shouldn't call super.
            return nil
        }
    }
    return super.layoutAttributesForItem(at: indexPath)
}
```

`.insertAtDestinationIndexPath` работает плохо, если тянуть ячейку из одной коллекции в другую. Приложение крашнется при драге за пределы первой секции, это связано с лейаутом. У таблиц проблем нет.

### Для `TableView`

Для таблицы есть аналогичные протоколы `UITableViewDragDelegate` и `UITableViewDropDelegate`. Методы повторяются с оговоркой на таблицу:

```swift
public protocol UITableViewDragDelegate: NSObjectProtocol {
    
    optional func tableView(_ tableView: UITableView, itemsForAddingTo session: UIDragSession, at indexPath: IndexPath, point: CGPoint) -> [UIDragItem]
    
    optional func tableView(_ tableView: UITableView, dragSessionWillBegin session: UIDragSession)
    
    optional func tableView(_ tableView: UITableView, dragSessionDidEnd session: UIDragSession)
}
```

Дроп работает без костылей в таблице, подозреваю что из-за отсутствия лейаута. Редактирование таблицы никак не влияет на вызовы методов дропа:

```swift
tableView.isEditing = true
```

То есть у вас может быть системный реордер ячеек и дроп внутрь ячеек.

[Перемещение и дроп ячейки из коллекции в таблицу.](https://cdn.sparrowcode.io/tutorials/drag-and-drop/table-drop.mov)

## `DestinationIndexPath`

Системный параметр `DestinationIndexPath` не всегда идеально определяет положение. Например, если вы выйдете за края контента коллекции, то система не предложит сбросить ячейку как последнюю.

Давайте напишем функцию, которая сможет предложить свой индекс, если системное предложение равно `nil`.

```swift
// В качестве входных параметров используем системный индекс и сессию дропа.
// Если системный индекс будет равен `nil`, то у нас появятся вторая системы прогноза индекса.

private func getDestinationIndexPath(system passedIndexPath: IndexPath?, session: UIDropSession) -> IndexPath? {
        
    // Здесь попытаемся получить индекс по локации дропа.
    // Чаще всего результат будет совпадать с системным, но когда системного нет, может вернуть хорошее значение.
    let systemByLocationIndexPath = collectionView.indexPathForItem(at: session.location(in: collectionView))
        
    // Здесь хардкор. Берём локацию и ищем в радиусе 100 точек ближайшую ячейку.
    var customByLocationIndexPath: IndexPath? = nil
    if systemByLocationIndexPath == nil {
        var closetCell: UICollectionViewCell? = nil
        var closetCellVerticalDistance: CGFloat = 100
        let tapLocation = session.location(in: collectionView)
            
        for indexPath in collectionView.indexPathsForVisibleItems {
            guard let cell = collectionView.cellForItem(at: indexPath) else { continue }
            let cellCenterLocation = collectionView.convert(cell.center, to: collectionView)
            let verticalDistance = abs(cellCenterLocation.y - tapLocation.y)
            if closetCellVerticalDistance > verticalDistance {
                closetCellVerticalDistance = verticalDistance
                closetCell = cell
            }
        }
            
        if let cell = closetCell {
            customByLocationIndexPath = collectionView.indexPath(for: cell)
        }
    }
        
    // Вернём значение в порядке приоритета.
    return passedIndexPath ?? systemByLocationIndexPath ?? customByLocationIndexPath
}
```

Улучшим код для обновления интерфейса:

```swift
func collectionView(_ collectionView: UICollectionView, dropSessionDidUpdate session: UIDropSession, withDestinationIndexPath destinationIndexPath: IndexPath?) -> UICollectionViewDropProposal {
        
    guard let _ = getDestinationIndexPath(system: destinationIndexPath, session: session) else { return .init(operation: .forbidden) }
    return .init(operation: .move, intent: .insertAtDestinationIndexPath)
}
```

Обратите внимание: метод поможет только с дропом. Если используете `.insertAtDestinationIndexPath`, не получится переопределить как будут расступаться ячейки.
