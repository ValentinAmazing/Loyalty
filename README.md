# Loyalty
Учебная система лояльности для банка с целью учёта начисления и выплаты кэшбэка за покупки, совершённые клиентами.

---------
### Description
У клиента банка может быть одна или несколько банковских карт (дополнительных), но только одна основная карта. Именно на неё осуществляется выплата вознаграждения.  
Анализируется период за предыдущий календарный месяц, кэшбэк выплачивается только тем, кто сделал хотя бы 10 расходных операций.  
Минимальная сумма кэшбека к выплате составляет 100 рублей, максимальная – 3000 руб.  
Лимит выплаты распространяется на все карты клиента.  
За покупки по карте в зависимости от пришедшего MCC начисляется повышенный процент кэшбэка, также повышенный процент начисления может быть у некоторых мерчантов (партнёров).  
За прочие покупки начисляется стандартный кэшбэк 1%, кроме определённого списка исключений MCCи мерчантов.  
Если транзакция попадает под действие нескольких условий для начисления (по MCCи мерчанту), то приоритет выбора процентной ставки кэшбэка следующий: список исключений MCC и мерчантов (0%), по мерчанту, по MCC, стандартная ставка (1%).

Клиент может совершать как обычные операции (покупки), так и возвраты. В случае возврата требуется уменьшать сумма кэшбэка к выплате пропорционально сумме возврата. Возможна обработка нескольких возвратов по одной покупке, но сумма возвратов не может превышать сумму оригинальной операции. Идентификатор оригинальной операции приходит вместе с возвратом. Требуется контролировать уникальность транзакций по идентификаторам прямых операций и возвратов. Не допустимо поступление операций после 10 числа текущего месяца за предыдущий.
Карты и клиентские данные в систему будут загружены при миграции. Требуется реализовать систему учёта транзакций, совершённых клиентами, с целью определения суммы кэшбэка к выплате. От банка будет поступать файл в CSV-формате, записи файла загружаются в таблицу файлового обмена. В ответ на пришедший файл система формирует ответный файл с результатами обработки: по каждой операции сумму начисленного кэшбэка, для возвратов – отрицательное число, а также актуальную сумму начислений на текущую дату (если бы клиенту выплатили кэшбэк сейчас). Эта информация будет использоваться для отображения клиенту банка в личном кабинете. Если запись исходного файла была отбита, ответный файл содержит по ней описание ошибки.

Также для АБС банка наша система должна уметь формировать конечный реестр начислений на отчётную дату (10 число текущего месяца) за прошедший отчётный период (1 календарный месяц). Реестр включает в себя номер основной карты и сумму кэшбэка к выплате. Не допустимо дублирование выгрузки реестра в банк.

---------
---------

### Структура входящего файла с транзакциями из банка
 Формат файла – CSV.  
 Разделитель полей – символ точки с запятой ";".

#### Заголовок
 Файл содержит ровно одну запись такого типа.  
 Всегда следует первой записью в файле.  
 
| Номер поля | Формат данных | Описание |
|:-:|:-:|:-|
| 1 | H (константа) | Тип записи |
| 2 | an..12 | Уникальный идентификатор файла в системе отправителя. <br/>Не допустима обработка файлов с одинаковыми идентификаторами |
| 3 | n14 | Дата и время формирования файла в формате YYYYMMDDHH24MISS |

#### Покупка
| Номер поля | Формат данных | Описание |
|:-:|:-:|:-|
| 1 | P (константа) | Тип записи |
| 2 | an40 | SHA1-криптограмма номера карты |
| 3 | an..12 | Уникальный идентификатор транзакции покупки в системе учёта мерчанта. <br/>Не допустима обработка транзакции с одинаковыми идентификаторами от одного мерчанта |
| 4 | n14 | Дата и время совершения покупки в формате YYYYMMDDHH24MISS |
| 5 | n..10 | Сумма покупки в минимальных единицах валюты RUB |
| 6 | an..30 | Идентификатор мерчанта |
| 7 | n4 | MCC |
| 8 | ans..2000 (опц.) | Дополнительное текстовое описание |

#### Возврат
| Номер поля | Формат данных | Описание |
|:-:|:-:|:-|
| 1 | R (константа) | Тип записи |
| 2 | an40 | SHA1-криптограмма номера карты. <br/>Может отличаться от карты, по которой была совершена покупка. <br/>Должна принадлежать тому же клиенту, что и карта покупки |
| 3 | an..12 | Уникальный идентификатор транзакции возврата в системе учёта мерчанта. <br/>Не допустима обработка транзакции с одинаковыми идентификаторами от одного мерчанта |
| 4 | n14 | Дата и время совершения возврата в формате YYYYMMDDHH24MISS |
| 5 | n..10 | Сумма возврата в минимальных единицах валюты RUB. <br/>По всем обработанным возвратам совокупно не допустимо превышение над суммой оригинальной покупки |
| 6 | an..30 | Идентификатор мерчанта |
| 7 | an..12 | Уникальный идентификатор транзакции покупки, для которой выполняется возврат. <br/>Возможно наличие нескольких возвратов для одной покупки |
| 8 | ans..2000 (опц.) | Дополнительное текстовое описание |

#### Концевик
Файл содержит ровно одну запись такого типа. Всегда следует последней записью в файле.

| Номер поля | Формат данных | Описание |
|:-:|:-:|:-|
| 1 | T (константа) | Тип записи |
| 2 | n..10 | Количество записей типа P |
| 3 | n..10 | Количество записей типа R |

-----


### Структура исходящего файла с результатами обработки транзакций в банк
Формат файла – CSV.  
Разделитель полей – символ точки с запятой ";".

#### Заголовок
 Файл содержит ровно одну запись такого типа.  
 Всегда следует первой записью в файле.

| Номер поля | Формат данных | Описание |
|:-:|:-:|:-|
| 1 | H (константа) | Тип записи |
| 2 | an..12 | Уникальный идентификатор файла в системе отправителя. <br/>Не допустима обработка файлов с одинаковыми идентификаторами |
| 3 | n14 | Дата и время формирования файла в формате YYYYMMDDHH24MISS |

#### Информация об успешной обработке транзакций
| Номер поля | Формат данных | Описание |
|:-:|:-:|:-|
| 1 | S (константа) | Тип записи |
| 2 | an40 | SHA1-криптограмма номера карты |
| 3 | an..12 | Уникальный идентификатортранзакции |
| 4 | n..10 | Размер кэшбэка по транзакции в минимальных единицах валюты RUB |
| 5 | n..10 | Размер кэшбэкак выплате на текущий момент в минимальных единицах валюты RUB |

#### Информация о неуспешной обработке транзакций
| Номер поля | Формат данных | Описание |
|:-:|:-:|:-|
| 1 | E (константа) | Тип записи |
| 2 | an40 | SHA1-криптограмма номера карты |
| 3 | an..12 | Уникальный идентификатор транзакции |
| 4 | n..5 | Код ошибки |
| 5 | ans..2000 | Текст ошибки |

#### Концевик
Файл содержит ровно одну запись такого типа. Всегда следует последней записью в файле.

| Номер поля | Формат данных | Описание |
|:-:|:-:|:-|
| 1 | T (константа) | Тип записи |
| 2 | n..10 | Количество записей типа S |
| 3 | n..10 | Количество записей типа E |

-----


### Структура исходящего файла с размером кэшбэка в банк
Формат файла – CSV.  
Разделитель полей – символ точки с запятой ";".

#### Заголовок
Файл содержит ровно одну запись такого типа.  
Всегда следует первой записью в файле.

| Номер поля | Формат данных | Описание |
|:-:|:-:|:-|
| 1 | H (константа) | Тип записи |
| 2 | an..12 | Уникальный идентификатор файла в системе отправителя. |
| 3 | n14 | Дата и время формирования файла в формате YYYYMMDDHH24MISS |
| 4 | n6 | Период начисления кэшбэка в формате YYYYMM |

### Информация о размере кэшбэка
| Номер поля | Формат данных | Описание |
|:-:|:-:|:-|
| 1 | C (константа) | Тип записи |
| 2 | an40 | SHA1-криптограмма номера карты |
| 3 | n..10 | Размер кэшбэка в минимальных единицах валюты RUB |

### Концевик
Файл содержит ровно одну запись такого типа.  
Всегда следует последней записью в файле.

| Номер поля | Формат данных | Описание |
|:-:|:-:|:-|
| 1 | T (константа) | Тип записи |
| 2 | n..10 | Количество записей типа C |


