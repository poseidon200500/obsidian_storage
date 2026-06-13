Сокращения:

|Сокращение|Значение|
|---|---|
|BASE|Базовое хранилище|
|INTERN|Интернирование|
|UV1|Unique V1: только `handle`|
|UV2|Unique V2: `handle + string`|

|Сценарий|Группа|Вставка|Память после GC|Память/unique|Сериализация|
|---|---|--:|--:|--:|--:|
|40% дублей, uniform|Распределение|BASE|BASE|BASE|BASE|
|40% дублей, Zipf|Распределение|BASE|BASE|BASE|BASE|
|10% дублей, uniform|% дубликатов|BASE|BASE|BASE|BASE|
|40% дублей, uniform|% дубликатов|BASE|BASE|BASE|BASE|
|80% дублей, uniform|% дубликатов|BASE|BASE|BASE|BASE|
|Длинные строки, до 12|Длина строк|BASE|BASE|BASE|BASE|
|Короткие строки, до 4|Длина строк|BASE|BASE|BASE|UV1|
|Средние строки, до 8|Длина строк|BASE|BASE|BASE|BASE|
|100 тыс. записей|Размер данных|BASE|BASE|BASE|UV1|
|1 млн записей|Размер данных|BASE|BASE|BASE|BASE|
|5 млн записей|Размер данных|BASE|BASE|BASE|BASE|
|Длинные строки, 95% дублей, uniform|Благоприятные для unique|BASE|UV1|UV1|UV2|
|Длинные строки, 95% дублей, Zipf|Благоприятные для unique|BASE|UV1|UV1|BASE|
|Очень длинные строки, 95% дублей, uniform|Благоприятные для unique|BASE|UV1|UV1|UV2|
|Очень длинные строки, 99% дублей, Zipf|Благоприятные для unique|BASE|UV1|UV1|BASE|