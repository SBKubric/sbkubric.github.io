### Movix Billing Service

Яндекс.Практикум, 25 Когорта, 7 команда

<br>

##### Состав команды

-   Станислав Богацкий, team lead, [github](https://github.com/sbkubric)
-   Виктор Гостяйкин, разработчик, [github](https://github.com/Viktor-Gostyaikin)
-   Сергей Колтунов, разработчик, [github](https://github.com/dogbusiness)

note: this is a not for a slide 1

h---

### Как всё начиналось...

План в моей голове:

![image](./content/start.png)

h---

### Хотелки бизнеса

![buisiness-requirements](./content/business_requirements.png)

h---

### А что делать?

![Мем с Траволтой](https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fmedia1.tenor.com%2Fimages%2Fc224be2073156574d456f8f3b86d927a%2Ftenor.gif%3Fitemid%3D14193582&f=1&nofb=1&ipt=e41c4371396a0e6909169de7a0817247d87712a7f3a2d709c66f9bc1d9e3ec0c&ipo=images)

[comment]: <> (Слайд для фразы "Для начала установим бизнес-требования и сформируем функциональные)
h---

### Cформируем список требований:

-   Интегрировать платежи в админку
-   Проводить платежи
-   Делать рефанды
-   Контролировать платеж
-   Легкая интеграция эквайринга без рефакторинга

h---

### Подписка

Я пользователь. Я хочу:

-   Я жму "Оплатить подписку"
-   Попадаю на форму с выбором метода оплаты
-   Ввожу платежные данные
-   Жму "оплатить"
-   Вижу статус платежа
-   Иду смотреть кино 🎥

h---

### Возврат

Я пользователь. Я хочу:

-   Я жму на кнопку "Отменить подписку"
-   Подтверждаю своё решение
-   Получаю письмо о возврате средств

h---

### Контроль платежей

Я администратор. Я хочу:

-   Видеть статус подписки у пользователя
-   Видеть информацию по платежам: время совершения транзакции, срок подписки, тип платежа

h---

[comment]: <> (Сервисы)

### Отлично, требования есть. А как делать будем?

![services](https://avatars.mds.yandex.net/i?id=71c84134dc73feeb9bb47bb24a798052a9361a07-10092505-images-thumbs&n=13)

h---

### Что нам вообще нужно?

![entities](./content/entities.png)

h---

### Кажется, одним сервисом дело не ограничится

![services](./content/services.svg)

h---

### Прикидываем архитектуру сервисов

![general](./content/general.svg)

h---

### А что с эквайрингом?

![cloudpayments](https://sovmart.ru/images/swjprojects/projects/2/ru-RU/cover.png)

h---

### Идемпотентность и автоматы

![automate](./content/automate.jpeg)

h---

### Библиотека transitions

<img src="./content/transitions.png" alt="drawing" width="50%" height="50%">

[pytransitions/transitions](https://github.com/pytransitions/transitions)

h---

### Начнем с данных

![data](https://avatars.mds.yandex.net/i?id=0ff0a7a78a4c6c7c209b89295a0ea563_l-4304125-images-thumbs&n=13)

h---

### Схема данных: биллинг

![payments-schema](./content/payments_schema.drawio.svg)

note: Сережа:
Привет. Как уже было сказано, меня зовутСергей и я один из разработчиков в этой команде.
Я хочу рассказать про схему данных для сервиса биллинга.

Пожалуй, главная таблица - Invoice. В ней хранятся все платежи, созданные subscription-api.
Здесь вопросы может вызвать только transaction_id - поле текстовое, потому что мы не знаем какой тип хранит у себя и отдает нам
тот или иной эквайринг. Мы решили оставаться гибкими и записывать айди в это универсальное поле.

Таблица рефанд существует для записи возвратов.
Возврат невозможен без существующего платежа, поэтому здесь есть FK для invoice_id.
Так же записывается transaction_id (возврат это ведь тоже транзакция у эквайринга)

Таблица acquiring_log предназначена для фиксации всех транзакций, проходящих в эквайринге.
Там мы записываем любую транзакцию и все возможные данные, получаемые от эквайринга:
номер транзакции, код который он нам вернул и сообщение.

h---

### Схема данных: подписки

![sub-schema](./content/sub_schema.drawio.svg)

h---

### Что происходит при подписке?

<img src="./content/payments_1.png" alt="drawing" width="70%" height="70%"/>

h---

### Детальные схемы хуков платежа

  <div style="display: flex; flex-direction: row;">
    <img src="./content/payments_2.png" alt="drawing" width="50%" height="50%"/>
    <img src="./content/payments_3.png" alt="drawing" width="50%" height="50%"/>
  </div>
h---

### Что происходит при возврате

<img src="./content/refunds_v2.drawio.svg" alt="drawing" width="78%" height="75%"/>

note: Сережа:
Я решил объединить вебхук от эквайринга и универсальный эндпоинт от нашего subscriptions-api в одну схему.

Сначала успешный сценарий:
От клиента передается запрос на возврат в subscriptions-api, затем по внутреннему эндпоинту в биллинг.
Биллинг находит платеж по которому хотят сделать возврат и создает запись в таблицу Refund. На этом же этапе проверяем есть ли уже возврат по этому платежу.
Если все хорошо, создаем платеж в статусе pending и отправляем эквайеру.
Если эквайер ответил чем-то кроме 200, фейлим платеж (меняем статус на фейлд) и заносим невыполненную транзакцию в acquiring_log.

h---

### Процесс биллинга

```python3
class BillingProcessABC(asyncio.AsyncMachine, mixins.TableMixin, abc.ABC):
    def __init__(
        self, *args, aquiring_provider: protocols.AcquiringProviderProtocol, **kwargs
    ):
        self._transaction_id: str | None = None
        self._status: str | None = None
        self._session: asql.AsyncSession | None = None
        self._locked_entry: db.Invoice | db.Refund | None = None
        self._entry_id: uuid.UUID | None = None
        self._acquiring_provider: protocols.AcquiringProviderProtocol = (
            aquiring_provider
        )
        super().__init__(*args, **kwargs)

    async def _before_change(self, event: transitions.EventData):
        if not self.table is None or (
            self._entry_id is None and self._transaction_id is None
        ):
            return

        self._session = db_session.get()
        if self._entry_id:
            result = await self._session.execute(
                sql.Select(self.table)
                .where(self.table.id == self._entry_id)
                .with_for_update()  # TODO : explore timeout possibilites
            )
            self._locked_entry = typing.cast(
                db.Invoice | db.Refund | None, result.one_or_none()
            )
            return

        result = await self._session.execute(
            sql.Select(self.table)
            .where(
                sql.and_(
                    self.table.aquiring_provider == self._acquiring_provider.title,
                    self.table.transaction_id == self._transaction_id,
                )
            )
            .with_for_update()
        )
        self._locked_entry = result.one_or_none()  # type: ignore

    async def get_pid(self) -> uuid.UUID:
        if self._entry_id is None:
            raise exc.NotFoundError
        return self._entry_id

    async def _after_change(self, event: transitions.EventData):
        if (
            self._entry_id is None
            or self._session is None
            or self._locked_entry is None
        ):
            return
        await self._session.commit()
        self._session = None
        self._locked_entry = None

    async def _store(self, event: transitions.EventData):
        if self._entry_id is None or self._locked_entry is None:
            raise exc.LogicError
        self._locked_entry.process = pickle.dumps(self)
        statement = sql.update(self.table).where(self.table.id == self._entry_id)
        await self._session.execute(statement, (self._locked_entry,))  # type: ignore

    @abc.abstractmethod
    async def on_exception(self, event: transitions.EventData):
        ...
```

h---

### Процесс платежа

```python
class InvoiceProcessABC(BillingProcessABC, mixins.InvoiceTableMixin, abc.ABC):
    def __init__(self, *args, **kwargs):
        states = enums.InvoiceStates
        self._locked_entry: db.Invoice | None = None
        self.table = self.__class__.table
        super().__init__(
            self,
            *args,
            states=states,
            initial=enums.InvoiceStates.NONE,
            send_event=True,
            before_state_change=self._before_change,
            after_state_change=self._after_change,
            **kwargs,
        )
        self.add_transition('store', list(enums.InvoiceStates), '=', after=self._store)
        self.add_transition(
            'create',
            enums.InvoiceStates.NONE,
            enums.InvoiceStates.CREATED,
            before=self._before_create,
            after=self._after_create,
        )
        self.add_transition(
            'check',
            enums.InvoiceStates.CREATED,
            enums.InvoiceStates.PENDING,
            before=self._before_check,
            after=self._after_check,
            conditions=self._is_locked,
        )
        self.add_transition(
            'pay',
            enums.InvoiceStates.PENDING,
            enums.InvoiceStates.PAID,
            before=self._before_pay,
            after=self._after_pay,
            conditions=self._is_locked,
        )
        self.add_transition(
            'cancel',
            [enums.InvoiceStates.CREATED, enums.InvoiceStates.PAID],
            enums.InvoiceStates.CANCELED,
            before=self._before_cancel,
            after=self._after_cancel,
            conditions=self._is_locked,
        )
        self.add_transition(
            'fail',
            [enums.InvoiceStates.CREATED, enums.InvoiceStates.PAID],
            enums.InvoiceStates.FAILED,
            before=self._before_fail,
            after=self._after_fail,
            conditions=self._is_locked,
        )

    def _is_locked(self) -> bool:
        return bool(self._locked_entry)

```

h---

### Процесс возврата

```python
class RefundProcessABC(mixins.RefundTableMixin, BillingProcessABC, abc.ABC):
    table = db.Refund

    def __init__(self, *args, **kwargs):
        states = enums.RefundStates
        self._locked_entry: db.Refund | None = None
        super().__init__(
            self,
            *args,
            states=states,
            initial=enums.RefundStates.NONE,
            send_event=True,
            before_state_change=self._before_change,
            after_state_change=self._after_change,
            **kwargs,
        )
        self.add_transition('store', list(enums.InvoiceStates), '=', after=self._store)
        self.add_transition(
            'create',
            enums.RefundStates.NONE,
            enums.RefundStates.CREATED,
            before=self._before_create,
            after=self._after_create,
        )
        self.add_transition(
            'register',
            enums.RefundStates.CREATED,
            enums.RefundStates.PENDING,
            before=self._before_register,
            after=self._after_register,
            conditions=[self._is_locked],
        )
        self.add_transition(
            'refunded',
            enums.RefundStates.PENDING,
            enums.RefundStates.REFUNDED,
            prepare=self._lock,
            before=self._before_refund,
            after=self._after_refund,
            conditions=self._is_locked,
        )
        self.add_transition(
            'fail',
            [enums.RefundStates.CREATED, enums.RefundStates.PENDING],
            enums.RefundStates.FAILED,
            prepare=self._lock,
            before=self._before_fail,
            after=self._after_fail,
            conditions=self._is_locked,
        )

    async def _find_acquirer(self, event: transitions.EventData):
        ...

    def _is_locked(self, *args, **kwargs) -> bool:
        return bool(self._locked_entry)
```

h---

[comment]: <> (Демо)

### А теперь смотрим!

[comment]: <> (![](content/my_video.mov))

[Демка биллинга](https://www.youtube.com/watch?v=dQw4w9WgXcQ&pp=ygUncmljayBhc3RsZXkgbmV2ZXIgZ29ubmEgZ2l2ZSB5b3UgdXAgb2xk)

h---

### Что не успели?

-   сервис подписок (готов только barebone и пара эндпойнтов)

h---

### Что можно улучшить?

-   рекуррентные платежи
-   Event-driven общение между сервисами

h---

### Вопросы?
