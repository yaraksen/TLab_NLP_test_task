# Запуск решения
Все решение находится в правильном порядке в Jupyter-ноутбуке, можно запускать подряд. Вместо заглушек необходимо вставить свои токены для API Hugging Face.

```python
API_TOKENS = ["<api-token1-from-hugging-face1", "<api-token1-from-hugging-face2", "..."]
```

# Отчет по тестовому заданию

## API Hugging Face
Для выполнения тестового задания предлагалось использовать распределенную модель Bloom. Так как Bloom Petals на протяжении двух недель выдавал ошибки о том, что не может найти пиров для части модели и заработал всего несколько раз, во всех экспериментах я использовал API Hugging Face, который дает доступ к той же самой модели BLOOM-176B. Так как в API стоит ограничение на количество запросов я написал функции, которые меняют токены и входят в режим ожидания, когда исчерпывается лимит по запросам.

## Датасет GSM8K
Состоит из школьных математических задач на английском языке и пошаговых решений к ним. Для извлечения ответа написал несколько функций, описание к ним в ноутбуке.

## Выборка для экспериментов
Так как API работает очень долго, не всегда стабильно и даже банит IP адреса, пришлось ограничить размер выборки до 200. Для полоноценных результатов и валидного сравнения метрики с другими моделями основные эксперименты надо провести на полной выборке, но для сравнения методов генерации в принципе хватает и 200.

## Few-shot prompting
В качестве примеров решения задач с chain of thought reasoning использовал промпты из приложения к [статье](https://arxiv.org/abs/2201.11903). С помощью них генерировал запросы к модели.

## Получение ответа из сгенерироованных решений
Так как необходимо было автоматизировать процесс подсчета точности решения моделью задач, необхоодимо было определить алгоритм определения правильного решения. Я попробовал несколько примеров и понял, что модель часто предсказывает потенциально верные решения, но немного не в том формате. Вот несколько примеров решения задач:

1. ```"Gary does laundry twice a week. Each load of laundry uses 20 gallons of water. So he uses 20 gallons of water twice a week. 20 gallons of water costs $0.15. So he spends $0.15 * 2 = $0.30 per week. So he spends $0.30 * 52 = $18.60 per year. The answer is $18.60.\n\nQ:"```
2. ```"Kalinda can add 4 pieces per minute. Her mom can add 2 pieces per minute. So Kalinda can add 4 * 60 = 240 pieces in an hour. Her mom can add 2 * 60 = 120 pieces in an hour. So Kalinda can add 240 pieces in an hour and her mom can add 120 pieces in an hour. So Kalinda can add 240 pieces in an hour and her mom can add 120 pieces in an hour. So Kalinda can add 240 + 120 = 360 pieces in an hour. 360 / 4 = 90 pieces per hour. So it will take them 90 * 4 = 360 hours to complete the puzzle"```
3. ```"The selling price is $350 000. The buyer has to pay 5% of the selling price, which is $12 500. Then he has to pay 12% of the selling price, which is $42 000. So the total price is $350 000 + $12 500 + $42 000 = $407 500. The answer is $407 500 - $400 000 = $7 500.\n\nA:"```

Я решил, что необязательно требовать от модели ответа строго в формате ```The answer is NUMBER```, поэтому дал ей небольшую свободу и за ответ принимал последнее число в ответе (стоит заметить, что я останавливал генерацию при встрече последовательностей ```['Q:', '\n\n', 'A:']```, поэтому ответ на задачу оказывался всегда в конце). Для того, чтобы вычленять из решений ответы и верно подсчитывать accuracy, используем регулярное выражение ```r"[-+]?(?:\d*\,?\ ?\.?\d*\,?\ ?\.?\d+)(?!.*\d)"```. Поддерживает числа вида 2,500, 42.5 и $7 500. Например:
```python
In: extract_model_answer("The selling price is $350 000. The buyer has to pay 5% of the selling price, which is $12 500. Then he has to pay 12% of the selling price, which is $42 000. So the total price is $350 000 + $12 500 + $42 000 = $407 500. The answer is $407 500 - $400 000 = $7 500.\n\nA:")
Out: 7500.0
```

## Эксперименты

### 1. Greedy search в качестве метода генерации решения
В качестве следующего токена на каждом шаге берем токен с максимальной вероятностью. Данный вариант использовался в статье [Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903). **Полученная accuracy составила 15.5%.**

### 2. Sampling в качестве метода генерации решения
Попробуем выбирать следующий токен случайным образом согласно полученному распределению. Решения должны получиться более случайными и скорее всего менее точными. Будем регулировать генерацию температурой софтмакса, которая чем меньше, тем более вырожденным становится распределение. При temp->0 должны получить генерацию практически идентичную greedy search.

При температуре равной 0.7 получили точность 8%, что почти в 2 раза меньше чем при жадном поиске, к тому же из-за случайности точность ответов не может гарантироваться. Можно сильно снизить случайность, если сделать температуру близкой к нулю, то есть жадный поиск с небольшой случайностью. Эксперимент с температурой 0.001 дал accuracy 16%, получили прирост 1%. Но стоит понимать, что так как сэмплинг это стохастический процесс, то из раза в раз могли получится результаты с разным accuracy. Логичным решением как раз является некоторое усреднение по нескольким случайным путям генерации, что по сути и является Self-Consistency, который протестируем ниже.

### 3. Beam Search в качестве метода генерации решения
Попробуем улучшить генерацию используя вместо обычного жадного поиска beam search c 10 лучами. Это позволит найти более вероятные последовательности, до которых не добирались простым жадным поиском,   рассмотривая разные возможные пути генерации (это делается параллелльно, поэтому на скорость не сильно повлияло).

#### <u>С ранней остановкой</u>
То есть если один возможный путь генерации достингет stop condition, то он сразу вернется моделью. **Удалось получить прирост в 0.5% качества, итого имеем 16.5%, но в данном случае этот результат уже не случайный, что лучше чем результаты сэмплинга при t=0.001**

#### <u>Без ранней остановки</u>
Точность упала до **14.5%**, возможно так как он генерировал более длинные, но при этом более вероятные последовательности с повторяющимися токенами, которые не приводили к верному ответу.

Попробуем улучшить результаты решения задач с помощью других способов случайной генерации.

### 4. Top-p Nucleus sampling
Обычный сэмплинг с какой то температурой вряд ли хорошо сработает при генерации решения математической задачи, так как нам не требуется разнообразность и неожиданность текста, как при генерации, например, рассказа. Поэтому попробуем генерировать используя некоторое ограничение на возможные токены. Будем распределять вероятность только среди минимального количества наиболее вероятных токенов, суммарная вероятность которых не меньше 0.9. Таким образом всякие нерелевентные токены отбросим, но при этом дадим некоторую свободу выбора модели. При **t=0.7 получил accuracy 11.5%**, при **t=1.0 accuracy равна 10.5%**. Результаты стали получше, чем у простого сэмплинга, но все равно ниже, чем у beam search. 

### 5. Contrastive search
Регулирует склонность модели генерировать повторяющиеся фразы, попробуем применить. **Accuracy получилась самая низкая среди всех способов: 7%.**

### 6. Self-Consistent Chain of Thoughts
Реализация метода предлолженного в статье [Self-Consistency Improves Chain of Thought Reasoning in Language Models](https://arxiv.org/abs/2203.11171).

Понятно, что метод генерации сэмплированием не может выдавать стабильную точность постоянно, поэтому, чтобы в некотором смысле уменьшить разброс решений, будем пытаться улучшить accuracy модели путем случайной генерации нескольких возможных решений и выбирая наиболее вероятный ответ. В статье пишут, что можно просто брать наиболее частый из сгенерированных ответов и accuracy окажется наибольшим, попробуем сделать так же, а не по прямой формуле считать веса каждого ответа в виде суммы вероятностей всех chain of thoughts которые сгенерировали каждый полученный ответ.

Для начала посмотрим с каким методом генерации для одинакового промпта генерятся разные, но при этом качественные ответы. Очевидно, что жадный детерминированный поиск рассматривать нет смысла, потому что мы хотим сгенерировать несколько разнообразных решений.

Рассмотрел следующую задачу ```'It takes Carmen 10 minutes to finish a crossword puzzle and 5 minutes to finish a sudoku puzzle.  Over the weekend she solved 3 crossword puzzles and 8 sudoku puzzles.  How much time did she spend playing these games?'```, где верный ответ 70, и с разными параметрами сгенеририровал по 10 решений и сравнил с какими параметрами получаются наилучшие результаты.

Наша цель генерировать разнообразные Chain of Thought, среди ответов которых верный наиболее распространенный. Получили неплохие результаты при temperature=0.7, где верный ответ встречается 4 раза из 10 и является самым частым, и при temperature=0.5, где 5 из 10 ответов верные. Попробовал использовать top-k, то есть на шаге выбора токена мы выбираем случайно только из 40 самых вероятных. При t=0.7 результаты оказались гораздо хуже и самый частый ответ неверный, но при t=0.5, так как там мы уже заранее сделали распределение токенов более вырожденным, верный ответ встречается чаще, но все равно реже чем без top-k. Top-p=0.9 также только ухудшил результаты. И t=0.001 большую часть раз выдавал неверный ответ, что еще раз подтверждает, что полученная выше accuracy 16% при t=0.001 могла быть гораздо меньше при повторении эксперимента несколько раз.

Проведем эксперимент с temperature=0.7, так как она дает одни из лучших результатов. Вот примеры сгенерировоанных верных и неверных решений для температуры равной 0.7:

- Carmen solved 3 crossword puzzles and 8 sudoku puzzles. 3 x 10 = 30 minutes and 8 x 5 = 40 minutes. So, 30 + 40 = 70 minutes. The answer is 70. ||| Extracted answer: 70.0

- Carmen started with 10 minutes. She spent this time on 3 crossword puzzles. 10 minutes x 3 crossword puzzles = 30 minutes. She also spent 10 minutes on 8 sudoku puzzles. So she spent 30 + 10 = 40 minutes. The answer is 40 minutes. ||| Extracted answer: 40.0

- Carmen started with 0 minutes. 3 crossword puzzles took 3 x 10 = 30 minutes. 8 sudoku puzzles took 5 minutes each, so that was 8 x 5 = 40 minutes. The answer is 30 + 40 = 70 minutes. ||| Extracted answer: 70.0

- Carmen spent 10 minutes on the crossword puzzle and 5 minutes on the sudoku puzzle. For the crossword puzzles, she spent 3 hours. 3 crossword puzzles * 10 minutes = 3 hours.  For the sudoku puzzles, she spent 8 hours. 8 sudoku puzzles * 5 minutes = 4 hours. So she spent 3 hours + 4 hours = 7 hours. The answer is 7 hours. ||| Extracted answer: 7.0

API HF Bloom не поддерживает аргумент генерации num_return_sequences, поэтому придется в цикле делать запросы. 
В ноутбуке я разделил выборку пополам, так как генерация для одной половины занимает около 6 часов, иначе Kaggle отключится. Так как на 200 запросов у меня уходит примерно 30 минут, то для эксперимента из статьи, где берут maximum vote на ответах по 40 сгенерированным решениям для каждой задачи, потребуется 20 часов (и то на выборке из 200 задач). Поэтому буду генерировать по 20 решений для каждой задачи вместо 40, чтобы уменьшить время работы. Petals ни в какую не работает, там можно было бы распараллелить инференс и векторизованно работать с данным, API Hugging Face не дает отправлять в Bloom батч запросов. Кроме того там ограничения около 200 запросов в час после чего надо ждать новые ресурсы, я постарался это обойти с помощью ротации токенов.

**Среди первой сотни вопросов 19 верно, среди второй сотни 26, то есть итоговая accuracy равна 22.5%**. Получили прирост в 6%.

В ноутбуке можно посмотреть массивы ответов для каждой из задач. Проанализировав и сравнив их с верными ответами можно заметить, что либо верный ответ встречается и является наиболее частым, либо он не встречается вовсе, либо он сгенерировался по количеству примерно наравне с каким то другим ответом, который оказался неверным. Из этого можно сделать вывод, что увеличение количества NUM_SEQUENCES точно улучшит accuracy, но при этом некоторые задачи в принципе плохо поддаются решению через BLOOM. Стоит посмотреть на эти типы задач и добавить аналогичные им в prompt.

Кроме того, в случае с GSM8K можно взять train выборку задач и разделить их, например, на 4 группы по логике решения, после чего сгенерировать 4 разных набора задач для  Few-shot prompting, каждый из которых будет покрывать некоторый ход решения. После этого запустить self-consistency chain-of-thought с NUM_SEQUENCES равным, например, 60, где каждая группа вопросов будет использоваться в качестве промпта 15 раз. После этого получим 60 ответов и выберем оттуда наиболее частый. Такой подход позволит модели концентрироваться на какой то одной логике решения и одна из них скорее всего подойдет, тогда модель сгенерирует много правильных ответов, а в остальных случаях множество ответов будет более разрозненным, поэтому наиболее частый вероятнее всего окажется верным.

Но для полного понимания надо провести много экспериментов на всей выборке, с разным количеством запросов для каждой группы промпта и параметрами генерации. В рамках тестового задания нет возможности это сделать, так как нет полноценного доступа к модели.

### Примеры сгенерированных во время экспериментов последовательностей можно посмотреть в [json файле](generated_examples.json). exp_name - короткое описание параметров генерации.

## Сравнения с другими моделями со схожим числом параметров
Так как я проводил эксперименты не на полной тестовой выборке GSM8K, то не совсем корректно сравнивать полученные результаты, но можем предположить, что выборка из 200 задач была достаточно репрезентативна.

[Статья 1](https://arxiv.org/abs/2201.11903)

[Статья 2](https://arxiv.org/abs/2203.11171)
```
CoT = Chain of Thought Prompting
SC = Self-Consistency
```

| Модель                                    | Количество параметров | GSM8K Accuracy (%) |
|-------------------------------------------|-----------------------|--------------------|
| Bloom (CoT)                               | 176B                  | 16.5               |
| Bloom (SC)                                | 176B                  | 22.5 (+6)          |
| LaMDA (CoT) (статья 1)                    | 137B                  | 14.3               |
| GPT-3 (CoT) (статья 1)                    | 175B                  | 46.9               |
| LaMDA (CoT) (статья 2)                    | 137B                  | 17.1               |
| LaMDA (SC) (статья 2)                     | 137B                  | 27.7 (+10.6)       |
| GPT-3 [Code-davinci-001] (CoT) (статья 2) | 175B                  | 14.6               |
| GPT-3 [Code-davinci-001] (SC) (статья 2)  | 175B                  | 23.4 (+8.8)        |
| GPT-3 [Code-davinci-002] (CoT) (статья 2) | 175B                  | 60.1               |
| GPT-3 [Code-davinci-002] (SC) (статья 2)  | 175B                  | 78.0 (+17.9)       |

Скорее всего если бы эксперименты проводились на полной выборке accuracy получился бы выше, но даже так модель сравнима с LaMDA, хотя у нее сильно меньше параметров, и с Code-davinci-001. Code-davinci-002 с большым разрывом обгоняет по точности решения задач все остальные модели с таким количеством параметров. Кроме того, по результатам экспериментов Bloom получила наименьший прирост при использовании Self-Consistency: 6% против 10.6% у LaMDA и 8.8% у Code-davinci-001.