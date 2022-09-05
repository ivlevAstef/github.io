---
layout: default
---

Сегодня случайно в очередной раз залез в код андройд команды. Надо было кое-что посмотреть. Стоит уточнить, что исторически так сложилось, что андройд проект существует у нас на 3 года дольше чем iOS, и поэтому в нем могут иногда содержатся различные гралли, с учетом, что с документацией требований в компании проблемы.

И нашел я вот такую там функцию:
```java
public Channel getAvailChannelForCurrentUser(boolean tryPrev) {
    User curUser = userManager.getCurrentUser();
    if (curUser == null) return null;

    //check current
    Channel curChannel = curUser.getChannel();
    User currentUser = userManager.getCurrentUser();
    if (curChannel != null
            && !curChannel.isEmergency()
            && curChannel.canEnter(currentUser)
            && !curChannel.isHidden(currentUser)) {
        Log.i(TAG, "getAvailChannelForCurrentUser user=" + curUser.getName() + " CURRENT channel=" + curChannel.getName());
        return curChannel;
    }

    //check PREVIOUS channel
    if (tryPrev) {
        curChannel = curUser.getChannelPrev();
        if (curChannel != null
                && !curChannel.isEmergency()
                && curChannel.canEnter(currentUser)
                && !curChannel.isHidden(currentUser)) {
            Log.i(TAG, "getAvailChannelForCurrentUser user=" + curUser.getName() + " PREV channel=" + curChannel.getName());
            return curChannel;
        }
    }

    //check LAST channel
    try {
        curChannel = getChannel(Integer.parseInt(UserKt.getLastChannel(curUser)));
    } catch (Exception e) {
        curChannel = null;
    }
    if (curChannel != null
            && !curChannel.isEmergency()
            && curChannel.canEnter(currentUser)
            && !curChannel.isHidden(currentUser)) {
        Log.i(TAG, "getAvailChannelForCurrentUser user=" + curUser.getName() + " LAST channel=" + curChannel.getName());
        return curChannel;
    }

    //check DEFAULT channel
    try {
        curChannel = getChannel(UserKt.getDefaultChannel(curUser));
    } catch (Exception e) {
        curChannel = null;
    }
    if (curChannel != null
            && !curChannel.isEmergency()
            && curChannel.canEnter(currentUser)
            && !curChannel.isHidden(currentUser)) {
        Log.i(TAG, "getAvailChannelForCurrentUser user=" + curUser.getName() + " DEFAULT channel=" + curChannel.getName());
        return curChannel;
    }

    //check ANY available
    for (Channel cChannel : getChannels()) {
        if (cChannel != null
                && !cChannel.isEmergency()
                && cChannel.canEnter(currentUser)
                && !cChannel.isHidden(currentUser)) {
            Log.i(TAG, "getAvailChannelForCurrentUser user=" + curUser.getName() + " ANY channel=" + cChannel.getName());
            return cChannel;
        }
    }
    return null;
}
```

Конечно эта функция не ужасная, и вполне себе нормальная, но в ней явно наблюдается нарушение принципа "не повторяйся" - то есть тут дублируется не просто куски кода (бог с ними), а дублируется одна и таже логика. Ну это основная проблема, есть и другие, давайте все проблемы вынесем в список:

- Дубликация логики - это я об if в 4 строки.
- Две одинаковых переменных currentUser и curUser.
- Переиспользование переменной curChannel. Мало того, что переиспользуется, так еще и название напрочь не верное.
- Длина функции такая, что не влазиет на один экран, из-за чего нельзя даже быстрым взглядом понять, что if то повторяется.

Название функции я не буду обсуждать - скажем так, оно вполне себе нормальное, вопрос вызывает только сокращение слова Available. Конечно нет придела совершенству, но в рамках того темпа разработки, что есть в компании название даже хорошее.

Я с вашего позволения, перепишу эту функцию на swift, так-как если я начну это упрощать на java, то могут полететь комментарии вида "а вот тут можно же сделать всё проще, если использовать вот такую конструкцию языка". Честно признаюсь, я в java умею писать условно говоря как на "C" - то есть помню только базовые конструкции языка. А так как сокращать код я буду не один раз, то лучше "не нарываться" :)

# Swift вариант эквивалент java

```swift
public func getAvailChannelForCurrentUser(_ tryPrev: Bool) -> Channel? {
    guard let curUser = userManager.currentUser else {
        return nil
    }

    //check current
    var curChannel: Channel? = curUser.channel
    let currentUser = userManager.currentUser!
    if let curChannel = curChannel,
       !curChannel.isEmergency
    && curChannel.canEnter(currentUser)
    && !curChannel.isHidden(currentUser) {
        Log.i("getAvailChannelForCurrentUser user=" + curUser.name + " CURRENT channel=" + curChannel.name)
        return curChannel
    }

    //check PREVIOUS channel
    if tryPrev {
        curChannel = curUser.prevChannel
        if let curChannel = curChannel,
           !curChannel.isEmergency
        && curChannel.canEnter(currentUser)
        && !curChannel.isHidden(currentUser) {
            Log.i("getAvailChannelForCurrentUser user=" + curUser.name + " PREV channel=" + curChannel.name)
            return curChannel
        }
    }

    //check LAST channel
    do {
        curChannel = getChannel(Int(UserKt.getLastChannel(curUser)))
    } catch {
        curChannel = nil
    }
    if let curChannel = curChannel,
       !curChannel.isEmergency
    && curChannel.canEnter(currentUser)
    && !curChannel.isHidden(currentUser) {
        Log.i("getAvailChannelForCurrentUser user=" + curUser.name + " LAST channel=" + curChannel.name)
        return curChannel
    }

    //check DEFAULT channel
    do {
        curChannel = getChannel(UserKt.getDefaultChannel(curUser))
    } catch {
        curChannel = nil
    }
    if let curChannel = curChannel,
       !curChannel.isEmergency
    && curChannel.canEnter(currentUser)
    && !curChannel.isHidden(currentUser) {
        Log.i("getAvailChannelForCurrentUser user=" + curUser.name + " DEFAULT channel=" + curChannel.name)
        return curChannel
    }

    //check ANY available
    for cChannel in getChannels() {
        if let curChannel = curChannel,
           !curChannel.isEmergency
        && curChannel.canEnter(currentUser)
        && !curChannel.isHidden(currentUser) {
            Log.i("getAvailChannelForCurrentUser user=" + curUser.name + " ANY channel=" + cChannel.name)
            return cChannel
        }
    }
    return nil
}
```

Я немного проявил небольшую своевольность и методы get{NAME} поменял на просто {NAME} так-как в языке swift скорей всего никто бы не писал гетеры и сеттеры как функции. При этом я пострался максимально точно сохранить первоначальный код. А да код я писал в блокноте, не пробовал собирать, поэтому есть шанс, что синтаксически я где-то ошибся.

# Рефакторинг

И так начнем с первого варианта рефакторинга:
```swift
public func getAvailChannelForCurrentUser(_ tryPrev: Bool) -> Channel? {
    guard let currentUser = userManager.currentUser else {
        return nil
    }

    func channelIsAvailable(_ channel: Channel) -> Bool {
        return channel.isEmergency 
            && channel.canEnter(currentUser) 
            && !channel.isHidden(currentUser)
    }

    //check current
    if let currentChannel = currentUser.channel, channelIsAvailable(currentChannel) {
        Log.i("getAvailChannelForCurrentUser user=" + currentUser.name + " CURRENT channel=" + currentChannel.name)
        return currentChannel
    }

    //check PREVIOUS channel
    if tryPrev {
        if let prevChannel = currentUser.prevChannel, channelIsAvailable(prevChannel) {
            Log.i("getAvailChannelForCurrentUser user=" + currentUser.name + " PREV channel=" + prevChannel.name)
            return prevChannel
        }
    }

    //check LAST channel
    if let lastChannel = try? getChannel(Int(UserKt.getLastChannel(currentUser))), channelIsAvailable(prevChannel) {
        Log.i("getAvailChannelForCurrentUser user=" + currentUser.name + " LAST channel=" + lastChannel.name)
        return lastChannel
    }

    //check DEFAULT channel
    if let defaultChannel = try? getChannel(UserKt.getDefaultChannel(currentUser)), channelIsAvailable(defaultChannel) {
        Log.i("getAvailChannelForCurrentUser user=" + currentUser.name + " DEFAULT channel=" + defaultChannel.name)
        return defaultChannel
    }

    //check ANY available
    for channel in getChannels() {
        if let channel = channel, channelIsAvailable(channel) {
            Log.i("getAvailChannelForCurrentUser user=" + currentUser.name + " ANY channel=" + channel.name)
            return channel
        }
    }
    return nil
}
```

Уже лучше - функция влезает на экран, все на своих местах, логика не дублируется. Идем дальше. Первое, что мы сделаем это избавимся от того, что мы перенесли с java, но с 99% вероятностью на swift такого бы не было - функция `getChannels()` скорей всего возвращает не `Array<Channel?>` а `Array<Channel>`, то есть в ней нет опциональных элементов.
```swift
//check ANY available
if let otherChannel = getChannels().first(where: channelIsAvailable) {
    Log.i("getAvailChannelForCurrentUser user=" + currentUser.name + " ANY channel=" + otherChannel.name)
    return otherChannel
}
```

Да я тут опять чуть схалявил, и сразу решил использовать возможности языка swift, дабы смысл цикла стал более наглядным - мы ищем первый элемент в массиве которые удовлетворяет критериям.

И вроде бы вот, вот оно всё идеально. И на самом деле в таком виде можно оставить функцию, и она читабельная, легко расширяемая, в ней нет дубликации логики. Но давайте еще поизвращаемся над кодом?

# Издевательства над кодом

Сейчас будет следующий виток развития этого кода. Не надо кричать, что он менее оптимальный и потеряно tryPrev, да еще и часть информации логов потерял - я и так знаю, но его стоит написать, чтобы переход был плавнее:
```swift
public func getAvailChannelForCurrentUser(_ tryPrev: Bool) -> Channel? {
    guard let currentUser = userManager.currentUser else {
        return nil
    }

    func channelIsAvailable(_ channel: Channel) -> Bool {
        return channel.isEmergency 
            && channel.canEnter(currentUser) 
            && !channel.isHidden(currentUser)
    }

    let priorityChannels: [Channel] = [
        currentUser.channel.flatMap { [$0] } ?? [],
        (currentUser.prevChannel.flatMap { [$0] } ?? [],
        (try? getChannel(Int(UserKt.getLastChannel(currentUser)))).flatMap { [$0] } ?? [],
        (try? getChannel(UserKt.getDefaultChannel(currentUser))).flatMap { [$0] } ?? [],
        getChannels()
    ].flatMap { $0 }

    guard let firstAvailableChannel = priorityChannels.first(where: channelIsAvailable) else {
        return nil
    }

    Log.i("getAvailChannelForCurrentUser user=" + currentUser.name + " channel=" + firstAvailableChannel.name)
    return firstAvailableChannel
}
```

Ого насколько короче. Давайте дам несколько разьяснений что произошло на этом шаге:

- Развернул if return. Зачем? теперь основное тело функции всегда находится на первом уровне вложенности, а ответвления просто делают возврат из функции.
- `.flatMap { [$0] }` Эта магическая чтука превращает опционал в случае если есть значением в массив из одного элемента. Ну а если нет, то остается nil который мы дальше с помощью `?? []` преобразуем в пустой массив.
- Создал массив всех каналов, где они находятся в том же порядке, что у нас было бы при использовании if. Но он создается магическим слегка образом за счет конструкции `.flatMap { $0 }`. flatMap разворачивает двумерный массив в одномерный, как-то вот так: `[[1,2,3],[4,5,6] -> [1,2,3,4,5,6]`, вторая удаляет все опциональные элементы/каналы.
- Убрал из логов указание "типа" канала.

Почему это будет работать дольше? Мы же не знаем насколько сложная например функция getChannel. И если в прошлом коде, до нее могло не дойти исполнение, то в этом коде, мы всегда получим все каналы, не зависимо ни от чего. В добавок вдруг "тип" канала в логах, это очень важная информация, которую нельзя упускать?

И это хоть и получилось коротко, но мы потеряли часть важной информации, еще и возможно создали лаги в программе.

Идем дальше:
```swift
private enum AvailableChannelType: CaseIterable {
    case current, prev, last, default, any
}
public func getAvailChannelForCurrentUser(_ tryPrev: Bool) -> Channel? {
    guard let currentUser = userManager.currentUser else {
        return nil
    }

    func channelIsAvailable(_ channel: Channel) -> Bool {
        return channel.isEmergency 
            && channel.canEnter(currentUser) 
            && !channel.isHidden(currentUser)
    }
    func availableChannel(_ channel: Channel?) -> Channel? {
        if let channel = channel, channelIsAvailable(channel) {
               return channel
           }
           return nil
    }
    func availableChannel(_ channelType: AvailableChannelType) -> Channel? {
        switch channelType {
        case .current: return availableChannel(currentUser.channel)
        case .prev: return availableChannel(tryPrev ? currentUser.prevChannel : nil)
           case .last: return availableChannel(try? getChannel(Int(UserKt.getLastChannel(currentUser))))
        case .default: return availableChannel(try? getChannel(UserKt.getDefaultChannel(currentUser)))
        case .any: return getChannels().first(where: channelIsAvailable)
        }
    }

    for channelType in AvailableChannelType.allCases {
        if let channel = availableChannel(channelType) {
            Log.i("getAvailChannelForCurrentUser user=" + currentUser.name + " " + channelType + " channel=" + channel.name)
            return channel
        }
    }
 
    return nil
}
```

Интересный, и думаю неожиданный вариант, не правда ли? :) Такой вариант не имеет проблем с точки зрения функционала - есть и проверка на tryPrev, есть и логи, и функции будут вызываться по мере их надобности. Вот ток строчек кода, он занимает не меньше, несмотря на другой стиль описания.

# Итоги по коду

Конечно код выше можно сократить, но думаю наврятли этот код станет от этого понятный. В прочем, если нам нужно сохранить прошлую производительность, и прошлое логирование, то не стоит так извращаться. Лучший и наиболее читабельный вариант, это с if-ами:
```swift
public func getAvailChannelForCurrentUser(_ tryPrev: Bool) -> Channel? {
    guard let currentUser = userManager.currentUser else {
        return nil
    }

    func channelIsAvailable(_ channel: Channel) -> Bool {
        return channel.isEmergency 
            && channel.canEnter(currentUser) 
            && !channel.isHidden(currentUser)
    }

    if let currentChannel = currentUser.channel, channelIsAvailable(currentChannel) {
        Log.i("getAvailChannelForCurrentUser user=" + currentUser.name + " CURRENT channel=" + currentChannel.name)
        return currentChannel
    }

    if tryPrev, let prevChannel = currentUser.prevChannel, channelIsAvailable(prevChannel) {
        Log.i("getAvailChannelForCurrentUser user=" + currentUser.name + " PREV channel=" + prevChannel.name)
        return prevChannel
    }

    if let lastChannel = try? getChannel(Int(UserKt.getLastChannel(currentUser))), channelIsAvailable(prevChannel) {
        Log.i("getAvailChannelForCurrentUser user=" + currentUser.name + " LAST channel=" + lastChannel.name)
        return lastChannel
    }

    if let defaultChannel = try? getChannel(UserKt.getDefaultChannel(currentUser)), channelIsAvailable(defaultChannel) {
        Log.i("getAvailChannelForCurrentUser user=" + currentUser.name + " DEFAULT channel=" + defaultChannel.name)
        return defaultChannel
    }

    if let otherChannel = getChannels().first(where: channelIsAvailable) {
        Log.i("getAvailChannelForCurrentUser user=" + currentUser.name + " ANY channel=" + otherChannel.name)
        return otherChannel
    }

    return nil
}
```

В добавок этот вариант имеет важное преимущество - если появится еще одно условие, с еще какой-то особенностью, оно легко впишется в этот код, в отличии от вариантов с массивом и enum-ом, которые сильно завязаны на то, что должен быть определенный формат данных.

И в коммерческом коде, я выбираю этот вариант. А вот если пробовать записать это максимально коротко, то получим:
```swift
public func getAvailChannelForCurrentUser(_ tryPrev: Bool) -> Channel? {
    if let currentUser = userManager.currentUser {
        func channelIsAvailable(_ channel: Channel) -> Bool {
            channel.isEmergency && channel.canEnter(currentUser) && !channel.isHidden(currentUser)
        }

        var priorityChannels: [Channel] = []
        priorityChannels += currentUser.channel.flatMap { [$0] } ?? []
        priorityChannels += tryPrev ? (currentUser.prevChannel.flatMap { [$0] } ?? []) : []
        priorityChannels += (try? getChannel(Int(UserKt.getLastChannel(currentUser)))).flatMap { [$0] } ?? []
        priorityChannels += (try? getChannel(UserKt.getDefaultChannel(currentUser))).flatMap { [$0] } ?? []
        priorityChannels += getChannels()

        if let firstAvailableChannel = priorityChannels.first(where: channelIsAvailable) {
            Log.i("getAvailChannelForCurrentUser user=" + currentUser.name + " channel=" + firstAvailableChannel.name)
            return firstAvailableChannel
        }
    }
    return nil
}
```

А теперь задайте себе вопрос - неужели нажать бездумного ctrl+C, ctrl+V несколько раз проще, чем выделить часть кода в функцию? Ведь если посмотреть начальный вариант и сравнить его с коммерчески пригодным, то основное отличие будет именно в этом - остальные изменения это так, мелочи.

А код изначально содержал сразу 4 if-а, и только один который с tryPrev добавлялся позже. При этом код вокруг этой функции меняли сотни раз, но ладно, тут можно списать на закон "работает не трож".

После этого тело проверки (логика) менялось один раз. И собственно говоря почему я обратил на эту функцию внимание - там снова должна поменяться логика. И уверяю - это снова будет копирование в 5 if-ов новой проверки.

# ИТОГИ

И как подводя итоги - я думаю почти всех в детстве пинали с фразой "уберись в комнате/зале/на парте" и т.п. И вроде бы казалось простая задача, но такая сложная. Но именно это простое правило которое на первый взгляд тратит ваше время, по факту экономит - ведь скорей всего вы и сами искали много раз какую-то вещь, из-за того, что не убрали её на место. 

Тут работает такая логика - убраться занимает некоторое время, но плоды уборки экономят время, больше чем заняла уборка.

В коде тоже самое - даже если так уж сложилось, что вы написав код/поирав в игрушки не смогли сразу-же сделать его читабельным/убраться, то кто же вам мешает убраться при следующем входе в файл/комнату?

Просто посчитаем затраты: чтобы переписать код на коммерческий вариант, на java (который я плохо знаю), мне потребовалось 5 минут. Чтобы прочитать эту функцию, разобраться в ней, и главное внутри каждого if-а осознать, что они все одинаковые, мне потребовалось 3 минуты, чтобы прочитать новую функцию, мне скорей всего потребуется максимум минута. С этим кодом работало 3 разработчки на протяжении 3 лет. Класс менялся около 30 раз, а сама функция используется в порядка 12 местах, тогда как вначале использовалась только в двух. Суммируем:

- 60 раз минимум эту функцию читали по быстрому, чтобы понять что она из себя представляет - на это мне ушло около 1.5минут.
- 10 раз минимум эту функцию анализировали, чтобы понять необходимость использования - это по времени 3 минуты.
- 200 раз минимум эту функцию просто пролистывали, и пытались понять, нужна ли она и тот ли кусок кода видно - это секунд 10.

Итого: 60 * 90 + 10 * 270 + 200 * 10 = 168минут было потрачено времени.
Если бы функция была убрана: 60 * 30 + 10 * 60 + 200 * 8 = 66минут + 10 минут на переписать. 76 минут.

Излишние затраты времени на какую-то одну функцию в самых минимальных подсчетах получились 1.5часа. Да копейки, а если таких функций в день ты видишь сотни? Да и считал я свою скорость восприятия информации. Она не самая быстрая в мире, но сильно выше среднего среди программистов. 

Все кто со мной знакомы в плане программирования (хотя и в повсеместных делах ситуация похожая), часто спрашивают "как ты умудряешься быть таким продуктивным?" - Один из ответов на этот вопрос именно в этом - я убираюсь за собой. А если уборка слишком дорогая, то ставлю красную ленточку вокруг места со свалкой, ну или выражаясь языком программирования пишу TODO. И да даже исправляю их, когда прохожу мимо этой ленточки.

P.S. По поводу восприятия информации. У людей после олимпиадного программирования чаще всего она хорошая, так-как там каждая секунда важна. Также она хорошая у игроков в компьютерные игры. Но за свою жизнь я повстречал одного человека, который не просто так числился в топ100 мира в олимпиадном программировании. Представьте себе простое условие задачи на 500-800 символов, и решение где-то на 200-400 символов кода. За сколько вы можете решить подобное? а хотябы осознать, как решать? А что если я вам скажу, что человек всё это делал за 5-10 секунд. Откройте codeforce и задачу A, или codingame и там clash of code, и попробуйте сдать решение верное за 5-10 секунд, от момента открытия. Скажу чесно - мой средний показатель это 1-2минуты. А у среднего коммерческого программиста около 5 и более минут. И кто-то скажет - ну просто много задач решил, вот и всё. Да опыт беспорно важен, но подобная скорость переносится на любые задачи. Но это так, ремарка.