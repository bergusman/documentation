---
title: Работа с чеком
keywords: sample
summary:
sidebar: evotordoc_sidebar
permalink: doc_receipt_interactions.html
folder: smart_terminal_SDK
---

### Добавление, изменение, удаление позиций в чеке

Вы можете взаимодействовать с чеком: добавлять, изменять и удалять позиции. Для этого требуется подписаться на событие редактирования позиций (`beforePositionsEditedEvent`), которое сообщает об изменении чека и содержит список изменений:

{% highlight java %}
public class BeforePositionsEditedEvent {
    private static final String TAG = "PositionsEditedEvent";

    public static final String NAME = "evo.v2.receipt.sell.beforePositionsEdited";
    private static final String KEY_CHANGES = "changes";

    public static BeforePositionsEditedEvent create(Bundle bundle) {
        Parcelable[] changesParcelable = bundle.getParcelableArray(KEY_CHANGES);
        return new BeforePositionsEditedEvent(
                Utils.filterByClass(
                        ChangesMapper.INSTANCE.create(changesParcelable),
                        IPositionChange.class
                )
        );
    }

    private List<IPositionChange> changes;

    public BeforePositionsEditedEvent(List<IPositionChange> changes) {
        this.changes = changes;
    }

    public Bundle toBundle() {
        Bundle result = new Bundle();
        Parcelable[] changesParcelable = new Parcelable[changes.size()];
        for (int i = 0; i < changesParcelable.length; i++) {
            IChange change = changes.get(i);
            changesParcelable[i] = ChangesMapper.INSTANCE.toBundle(change);
        }
        result.putParcelableArray(KEY_CHANGES, changesParcelable);
        return result;
    }

    public List<IPositionChange> getChanges() {
        return changes;
    }
}
{% endhighlight %}

#### Список изменений

Изменение сообщает о том, что будет добавлена позиция:

{% highlight java %}
data class PositionAdd(val position: Position) : IPositionChange {

    override fun toBundle(): Bundle {
        return Bundle().apply {
            putBundle(
                    PositionMapper.KEY_POSITION,
                    PositionMapper.toBundle(position)
            )
        }
    }

    override fun getPositionUuid(): String? {
        return position.uuid
    }

    override fun getType(): IChange.Type {
        return IChange.Type.POSITION_ADD
    }

    companion object {
        @JvmStatic
        fun from(bundle: Bundle?): PositionAdd? {
            bundle ?: return null

            return PositionAdd(
                    PositionMapper.from(
                            bundle.getBundle(PositionMapper.KEY_POSITION)
                    ) ?: return null
            )
        }
    }
}
{% endhighlight %}

Изменение сообщает том, что позиция будет отредактирована:

{% highlight java %}
data class PositionEdit(val position: Position) : IPositionChange {

    override fun toBundle(): Bundle {
        return Bundle().apply {
            putBundle(
                    PositionMapper.KEY_POSITION,
                    PositionMapper.toBundle(position)
            )
        }
    }

    override fun getPositionUuid(): String? {
        return position.uuid
    }

    override fun getType(): IChange.Type {
        return IChange.Type.POSITION_EDIT
    }

    companion object {
        @JvmStatic
        fun from(bundle: Bundle?): PositionEdit? {
            bundle ?: return null

            return PositionEdit(
                    PositionMapper.from(bundle.getBundle(PositionMapper.KEY_POSITION)) ?: return null
            )
        }
    }
}
{% endhighlight %}

Изменение сообщает том, что позиция будет удалена:

{% highlight java %}
data class PositionRemove(
        private val positionUuid: String
) : IPositionChange {

    override fun toBundle(): Bundle {
        return Bundle().apply {
            putString(
                    KEY_POSITION_UUID,
                    positionUuid
            )
        }
    }

    override fun getPositionUuid(): String? {
        return positionUuid
    }

    override fun getType(): IChange.Type {
        return IChange.Type.POSITION_REMOVE
    }

    companion object {
        const val KEY_POSITION_UUID = "positionUuid"

        @JvmStatic
        fun from(bundle: Bundle?): PositionRemove? {
            bundle ?: return null

            return PositionRemove(
                    bundle.getString(KEY_POSITION_UUID) ?: return null
            )
        }
    }
}
{% endhighlight %}

#### Подписка на событие

1. Создайте службу, например `MyIntegrationService`, которая наследует класс `IntegrationService`. В колбэке `onCreate` службы, зарегистрируйте процессор `BeforePositionsEditedEventProcessor` (процессор наследует класс `ActionProcessor`).
    {% highlight java %}
    public class MyIntegrationService extends IntegrationService {
        @Override
        public void onCreate() {
            registerProcessor(new BeforePositionsEditedEventProcessor() {
                @Override
                public void call(BeforePositionsEditedEvent beforePositionsEditedEvent, Callback callback) {
                       });
                    }
                }
    {% endhighlight %}
2. Объявите службу в манифесте приложения:
    {% highlight java %}
    <service
            android:name=".MyIntegrationService"
            android:enabled="true"
            android:exported="true">
            <intent-filter>
                <category android:name="android.intent.category.DEFAULT" />
                <action android:name="evo.v2.receipt.sell.beforePositionsEdited" />
            </intent-filter>
    </service>
    {% endhighlight %}

В метод `call` процессора приходит событие `beforePositionsEditedEvent` и объект для возврата результата `callback`.



В ответ изменения могут вернуть результат со списком дополнительных изменений:

{% highlight java %}
public class BeforePositionsEditedEventResult {

    private static final String KEY_RESULT = "result";
    private static final String KEY_CHANGES = "changes";

    public static BeforePositionsEditedEventResult create(Bundle bundle) {
        String resultName = bundle.getString(KEY_RESULT);
        Parcelable[] changesParcelable = bundle.getParcelableArray(KEY_CHANGES);
        List<IChange> changes = ChangesMapper.INSTANCE.create(changesParcelable);
        List<IPositionChange> positionChanges = new ArrayList<>();
        for (IChange change : changes) {
            if (change instanceof IPositionChange) {
                positionChanges.add((IPositionChange) change);
            }
        }
        return new BeforePositionsEditedEventResult(
                Utils.safeValueOf(Result.class, resultName, Result.UNKNOWN),
                positionChanges
        );
    }

    private final Result result;
    private final List<IPositionChange> changes;

    public BeforePositionsEditedEventResult(Result result, List<IPositionChange> changes) {
        this.result = result;
        this.changes = changes;
    }

    public Bundle toBundle() {
        Bundle bundle = new Bundle();
        bundle.putString(KEY_RESULT, result.name());
        Parcelable[] changesParcelable = new Parcelable[changes.size()];
        for (int i = 0; i < changesParcelable.length; i++) {
            IChange change = changes.get(i);
            changesParcelable[i] = ChangesMapper.INSTANCE.toBundle(change);
        }
        bundle.putParcelableArray(KEY_CHANGES, changesParcelable);
        return bundle;
    }

    public Result getResult() {
        return result;
    }

    public List<IPositionChange> getChanges() {
        return changes;
    }

    public enum Result {
        OK,
        UNKNOWN;
    }
}
{% endhighlight %}

Чтобы вернуть результат, используйте метод:
{% highlight java %}
callback.onResult(beforePositionsEditedEventResult.toBundle())
{% endhighlight %}


Если приложению для возврата результата необходимо взаимодействие с пользователем, запустите операцию (`Activity`), которая наследует класс `BeforePositionsEditedEventActivity`:
{% highlight java %}
callback.startActivity(new Intent(MyIntegrationService.this, MainActivity.class));
{% endhighlight %}

Ваша операция должна вызвать метод
`setIntegrationResult`.

Например:
{% highlight java %}
setIntegrationResult(new BeforePositionsEditedEventResult(BeforePositionsEditedEventResult.Result.OK, changes));
{% endhighlight %}

Класс `BeforePositionsEditedEventActivity` задан как:
{% highlight java %}
public class BeforePositionsEditedEventActivity extends IntegrationActivity {
    public void setIntegrationResult(BeforePositionsEditedEventResult result) {
        setIntegrationResult(result == null ? null : result.toBundle());
    }

    public BeforePositionsEditedEvent getEvent() {
        return BeforePositionsEditedEvent.create(getSourceBundle());
    }
}
{% endhighlight %}