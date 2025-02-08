# Отчёт по задаче коммивояжёра

## Содержание
- [Теоретический обзор](#1-теоретический-обзор)
    - [Постановка задачи](#11-постановка-задачи)
    - [Современные подходы](#12-современные-подходы)
- [Мой подход и глубокий анализ решения](#2-мой-подход-и-глубокий-анализ-решения)
- [Результаты-тестирования](#3-результаты-тестирования)
- [Выводы](#4-выводы)
- [Заключение](#5-заключение)
- [Мои статьи по машинному обучению](#6-мои-статьи-по-машинному-обучению)

## 1. Теоретический обзор

### 1.1. Постановка задачи

Формально задача коммивояжёра формулируется так:  
> Дан граф с `n` вершинами и положительными весами на рёбрах. Необходимо найти гамильтонов цикл (маршрут, проходящий через каждую вершину ровно один раз с возвращением в исходную) минимальной суммарной длины.

Как известно, число возможных маршрутов в симметричном случае равно `(n-1)!/2`, что делает задачу неразрешимой перебором уже при относительно небольшом `n`. Существуют точные методы, такие как динамическое программирование (алгоритм Беллмана–Хелда–Карпа) и метод ветвей и границ, а также эвристические методы (жадные алгоритмы, имитация отжига, генетические алгоритмы, муравьиный алгоритм). Последние часто применяются для получения «достаточно хороших» приближённых решений на графах большой размерности.

### 1.2. Современные подходы

В литературе (например, в работах [Kool, van Hoof, Welling](https://arxiv.org/pdf/1803.08475) и др.) активно применяются методы глубокого обучения, где используется encoder–decoder архитектура с механизмом внимания для генерации маршрутов. Такие методы демонстрируют возможность получения решений, отклоняющихся от оптимума менее чем на 2–3 %, и способны масштабироваться при использовании GPU.

При этом классические эвристические методы, такие как алгоритм ближайшего соседа, остаются востребованными из-за своей простоты и быстродействия, хотя они часто дают решения, отличающиеся от оптимальных на 10–30 % и более. Методы динамического программирования и ветвей и границ гарантируют оптимальность, но их экспоненциальная сложность ограничивает применение на больших графах.

---

## 2. Мой подход и глубокий анализ решения

В [моём решении](https://github.com/ALeksandr13254/TSP-with-ML/blob/main/neural-tsp.ipynb) я комбинировал идею обучения параметров переходов между вершинами с классическими эвристиками построения маршрута. Основные этапы моего алгоритма таковы:

1. **Чтение входных данных и формирование матрицы смежности.**  
   Функция `read_atsp_file(filename)` читает данные из файла формата ATSP и преобразует их в матрицу весов.

2. **Обучение модели.**  
   Функция `train_model(distance_matrix, penalty=3.0, epochs=1000, lr=0.05)` выполняет оптимизацию матрицы весов с использованием оптимизатора Adam. Здесь я нормализую матрицу расстояний, обнуляю диагональ и обучаю параметры так, чтобы итоговый softmax-выход можно было интерпретировать как вероятностное распределение переходов между вершинами. Это распределение далее используется при построении маршрутов.

3. **Построение маршрута.**  
   Я реализовал два способа построения маршрута:
   - **Жадный алгоритм** (`greedy_path`): стартуя из заданной вершины, на каждом шаге выбирается вершина с максимальной вероятностью перехода (из полученного распределения).  
   - **Вероятностный (стохастический) алгоритм** (`probabilistic_path`): с учетом порогового значения вероятности происходит случайный выбор следующей вершины, что позволяет исследовать альтернативные маршруты и зачастую дает решение с меньшим отклонением от оптимума.

4. **Вычисление длины маршрута.**  
   Функция `calculate_route_length(path, distance_matrix)` суммирует веса выбранных рёбер, при необходимости замыкая цикл, если маршрут не является замкнутым.

5. **Объединение графов.**  
   Функция `combine_graphs(graphs, optimal_lengths)` позволяет объединить несколько тестовых графов в один большой, что особенно полезно для тестирования алгоритма на более крупных данных.

Таким образом, мой алгоритм сначала «обучается» на матрице расстояний, после чего генерирует маршруты с помощью жадного и стохастического подходов. Для повышения качества решения я запускаю случайный поиск несколько раз, выбирая маршрут с минимальной длиной.

---

## 3. Результаты тестирования

Ниже приведена таблица результатов проверки моего метода на стандартных тестовых файлах с ациклическими графами (ATSP):

| Файл            | Оптимум | Жадный     | Случайный | Отклонение жадного (%) | Отклонение случайного (%) |
|-----------------|---------|------------|-----------|------------------------|---------------------------|
| br17.atsp       | 39      | 39         | 39        | 0.0%                   | 0.0%                      |
| ft53.atsp       | 6905    | 7967       | 7543      | 15.4%                  | 9.2%                      |
| ft70.atsp       | 38673   | 40410      | 39455     | 4.5%                   | 2.0%                      |
| ftv33.atsp      | 1286    | 1616       | 1474      | 25.7%                  | 14.6%                     |
| ftv35.atsp      | 1473    | 1893       | 1705      | 28.5%                  | 15.8%                     |
| ftv38.atsp      | 1530    | 1942       | 1763      | 26.9%                  | 15.2%                     |
| ftv44.atsp      | 1613    | 1990       | 1964      | 23.4%                  | 21.8%                     |
| ftv47.atsp      | 1776    | 2080       | 1984      | 17.1%                  | 11.7%                     |
| ftv55.atsp      | 1608    | 2044       | 1915      | 27.1%                  | 19.1%                     |
| ftv64.atsp      | 1839    | 2038       | 2004      | 10.8%                  | 9.0%                      |
| ftv70.atsp      | 1950    | 2418       | 2306      | 24.0%                  | 18.3%                     |
| kro124p.atsp    | 36230   | 43499      | 41284     | 20.1%                  | 13.9%                     |
| p43.atsp        | 5620    | 5723       | 5656      | 1.8%                   | 0.6%                      |
| rbg323.atsp     | 1326    | 1375       | 1358      | 3.7%                   | 2.4%                      |
| rbg358.atsp     | 1163    | 1314       | 1232      | 13.0%                  | 5.9%                      |
| rbg403.atsp     | 2465    | 2707       | 2503      | 9.8%                   | 1.5%                      |
| rbg443.atsp     | 2720    | 3029       | 2776      | 11.4%                  | 2.1%                      |
| ry48p.atsp      | 14422   | 17364      | 16382     | 20.4%                  | 13.6%                     |
| **combined2252**| 308398  | 314251     | 312281    | 1.9%                   | 1.3%                      |

---

## 4. Выводы

В результате были получены следующие выводы:

- **Обучение модели:** Использование метода обучения на матрице расстояний позволило сформировать вероятностное распределение переходов, которое в свою очередь использовалось для построения маршрута. Это даёт гибкость при выборе между жадным и стохастическим подходом.
- **Качество решений:** На малых графах (br17.atsp) алгоритм находит оптимальное решение, а на более крупных тестовых экземплярах (ft53.atsp, ft70.atsp, и т.д.) наблюдаются отклонения жадного алгоритма в пределах 15–30 %, а случайного поиска – обычно ниже (2–15 %). При объединении графов отклонение значительно уменьшается, что говорит о том, что при больших размерностях мой подход демонстрирует высокую стабильность.
- **Практическая применимость:** Подход позволяет решать задачу коммивояжёра на графах с тысячами вершин за приемлемое время, а использование PyTorch открывает возможность масштабирования и параллельных вычислений (например, на GPU).

Таким образом, мой алгоритм, сочетающий обучение модели для формирования переходных вероятностей и классические эвристические методы построения маршрута, является эффективным для практического применения в реальных задачах комбинаторной оптимизации.

---

## 5. Заключение

В данном отчёте описана теоретическая база задачи коммивояжёра, современные методы её решения, подробно разобрано своё решение с использованием обучения модели и эвристик, а также приведены результаты тестирования на ряде стандартных ациклических графах. Полученные результаты показывают, что подход успешно справляется как с небольшими, так и с крупными графами, демонстрируя высокую точность (в ряде случаев отклонение менее 2 %) и стабильность решения.

В дальнейшем планируется работа по адаптации алгоритма для варианта задачи с учётом дополнительных затрат (например, поворотов), что позволит применять его в более сложных условиях, характерных для городских дорожных сетей и логистических систем.

---

## 6. Мои статьи по машинному обучению
Также хотелось бы поделиться своими исследованиями, отраженными в следующих статьях журналов из перечня ВАК и РИНЦ.
1. Астраханцева, И., Герасимов, А., &amp; Смирнова, О. (2024). Оценка применимости статистических и машинных моделей для прогнозирования инфляции. Современные наукоёмкие технологии. Региональное приложение, 79(3), 120-131. извлечено от http://snt-isuct.ru/article/view/6049
2. Астраханцева И.А., Герасимов А.С. Прогнозирование региональной инфляции на основе гибридной модели машинного обучения: градиентный бустинг и случайный лес НАУЧНЫЕ ТРУДЫ Вольного экономического общества России – 2023. – № 5. – С. 200-226. – DOI: 10.38197/2072-2060-2023-243-5-200-226 https://cyberleninka.ru/article/n/prognozirovanie-regionalnoy-inflyatsii-na-osnove-gibridnoy-modeli-mashinnogo-obucheniya-gradientnyy-busting-i-sluchaynyy-les
3. Астраханцева И.А., Герасимов А.С., Астраханцев Р.Г. Прогнозирование региональной инфляции с помощью алгоритмов машинного обучения. Известия высших учебных заведений. Серия: Экономика, финансы и управление производством. – 2022. – № 4(54). – С. 6–13. DOI 10.6060/ivecofin.2022544.620. – EDN ITYDFE. https://cyberleninka.ru/article/n/prognozirovanie-regionalnoy-inflyatsii-s-pomoschyu-algoritmov-mashinnogo-obucheniya
4. Герасимов А.С., Бобкова С.П. Применение машинного обучения для прогнозирования результатов подготовки студентов. Сборник научных трудов вузов России «Проблемы экономики, финансов и управления производством» - 2022.- № 50 – С.56-62 УДК: 004.94 (РИНЦ 2018) https://elibrary.ru/item.asp?edn=hrumrz
