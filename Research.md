
# research.md — журнал поиска и решений

## 1. Какие датасеты рассматривались

| # | Источник | Содержание | Итог |
|---|---|---|---|
| 1 | Kaggle: Multi-Organism Bacterial Imaging Dataset | 10 организмов × 2 типа съёмки («gram stain» — микроскопия, «media plate» — колонии на чашке), 629 изображений, 640×640 | Использован, но только микроскопия («gram stain»). Есть грибы (Aspergillus, Candida) https://www.kaggle.com/datasets/ujjwalsinha01/multi-organism-bacterial-imaging-dataset|
| 2 | Kaggle-зеркало DIBaS (Zieliński et al., 2017) | много видов, окраска по Граму, микроскопия; размер архива ~4 ГБ | Использован; не более 30 картинок на вид. В пуле: 689 картинок, 33 видов https://www.kaggle.com/datasets/samaarashidaarbi/dibas-bacterial-colony-dataset?resource=download |
| 3 | Kaggle-набор «foodborne pathogens» (600 фото, по 150 на род Escherichia, Listeria, Salmonella, Staphylococcus; описан в PeerJ CS) | по описанию подходил лучше всего | Не использован: не удалось найти точную страницу набора и проверить содержимое. |
| 4 | Hallström et al., 2023 (Zenodo, таймлапсы микрофлюидики) | много кадров одних и тех же ловушек | Не использован: сложный формат и высокий риск утечки между кадрами |
| 5 | Roboflow Universe | https://universe.roboflow.com/ta-fhjnu/bacteria-weceh Использовался для проототипа, pip install в начале применялся, потом замениои метод.| Формат изображений не с микроскопа, а чашка Петри|

## 2. Принятые решения

- Только микроскопия: папки «media plate» исключены, так как это фотографии колоний, а не микроскопические изображения.
- Два независимых источника. Никакие данные не смешивались «по классам» (E. coli из одного набора, остальное из другого), чтобы модель не училась отличать набор, а не бактерию. В каждом источнике есть оба класса.
- Разбиение по группам дубликатов и стратификация по источнику и виду; параметры и порог Uncertain подбираются только по val.
- Основная метрика — recall и precision по E. coli с доверительными интервалами, а не accuracy.

## 3. Проверка данных (числа из запуска)

- Картинок в пуле: 1011; E. coli: 50.
- Групп почти-дубликатов: 11 (картинок в них: 22); групп с обоими классами было 3; удалено идентичных копий с неверной меткой: 3.
- «Читерский» тест (AUC по метаданным): dibas: 0.96, multi: 0.64.
- Пересечение DIBaS и multi по хэшу: 0 картинок.
- Различия источников по простым характеристикам снимков:

| source | width | height | brightness | contrast | sharpness | border_brightness |
|---|---|---|---|---|---|---|
| dibas | 2048.0 | 1532.0 | 193.4 | 19.1 | 236.4 | 193.4 |
| multi | 640.0 | 640.0 | 175.1 | 39.0 | 791.3 | 174.3 |

## 4. Ход экспериментов

1. **Просмотр данных.** Набор multi содержит фото чашек (media plate) рядом с микроскопией (gram stain); чашки исключены, остаются 325 микроскопических изображений, из них 30 — E. coli.
2. **Первая попытка обучения** (только источник multi, ResNet-18, сразу дообучается вся сеть, 12 эпох): потери на обучении падали (0.67 → 0.17), а качество на валидации не росло (AUC 0.59-0.74) — переобучение при ~20 примерах E. coli в обучении.
3. **Смена данных:** добавлен второй источник (DIBaS), чтобы увеличить число E. coli и получить независимую проверку.
4. **Смена обучения:** сначала только последний слой, затем аккуратное дообучение всей сети с малым lr (`finetune`) в сравнении с вариантом `head_only`; выбор эпохи по среднему AUC и balanced accuracy на val.

| Вариант | Параметры | val AUC на лучшей эпохе | Балл выбора | Temperature | tau |
|---|---|---|---|---|---|
| head_only | {'epochs_head': 12, 'epochs_ft': 0} | 0.820 | 0.780 | 0.70 | 0.8 |
| finetune | {'epochs_head': 3, 'epochs_ft': 9, 'lr_ft': 3e-05} | 0.911 | 0.902 | 0.95 | 0.8 |

Выбран вариант `finetune`.

## 5. Итоговые метрики

| Набор | N (из них E. coli) | Recall E. coli (95% ДИ) | Precision E. coli | Balanced acc (95% ДИ) | ROC-AUC | PR-AUC | Принято без Uncertain | E. coli ушло в Uncertain |
|---|---|---|---|---|---|---|---|---|
| ВНЕШНИЙ: DIBaS (обучение только на multi) | 689 (20) | 1.00 [1.0; 1.0] | 0.03 | 0.58 [0.56; 0.59] | 0.40 | 0.02 | 0 % | 20 из 20 |



## 6. Что пошло не так по ходу работы

- Набор, рекомендованный AI-ассистентом на первом шаге (600 фото, 4 рода), найти не удалось; фактически использованный набор оказался другим (10 организмов × 2 типа съёмки), в нём была часть фото чашек. Это было замечено при просмотре структуры папок и исправлено.
- Основная трудность была связана с поиском подходящего датасета. Большая часть найденных наборов содержала изображения чашек Петри, колоний или была предназначена для object detection, тогда как для задачи нужны были именно микроскопические изображения бактерий.

Также пришлось учитывать различия между источниками данных: фон, окраску, увеличение, плотность бактерий и качество изображений. Это стало особенно заметно при независимой проверке — модель показала более слабый результат, чем на внутреннем test.

-Отдельной проблемой оказался дисбаланс классов: E. coli в некоторых найденных наборах представлена значительно меньшим количеством изображений, чем другие бактерии.

-При работе с моделью возникали технические ошибки в коде и несовпадения названий переменных, которые пришлось исправлять при последовательном запуске Colab. После этого удалось получить рабочий pipeline: загрузка данных → обучение → оценка → классификация нового изображения → вывод E. coli / Not E. coli / Uncertain.

В результате стало понятно, что для такой задачи качество и разнообразие данных не менее важны, чем выбор самой нейросети.



## 7. Виды в пуле

| source | species | картинок |
|---|---|---|
| dibas | acinetobacter.baumanii | 20 |
| dibas | actinomyces.israeli | 23 |
| dibas | bacteroides.fragilis | 23 |
| dibas | bifidobacterium.spp | 23 |
| dibas | candida.albicans | 20 |
| dibas | clostridium.perfringens | 23 |
| dibas | enterococcus.faecalis | 20 |
| dibas | enterococcus.faecium | 20 |
| dibas | escherichia.coli | 20 |
| dibas | fusobacterium | 23 |
| dibas | lactobacillus.casei | 20 |
| dibas | lactobacillus.crispatus | 20 |
| dibas | lactobacillus.delbrueckii | 20 |
| dibas | lactobacillus.gasseri | 20 |
| dibas | lactobacillus.jehnsenii | 20 |
| dibas | lactobacillus.johnsonii | 20 |
| dibas | lactobacillus.paracasei | 20 |
| dibas | lactobacillus.plantarum | 20 |
| dibas | lactobacillus.reuteri | 20 |
| dibas | lactobacillus.rhamnosus | 20 |
| dibas | lactobacillus.salivarius | 20 |
| dibas | listeria.monocytogenes | 22 |
| dibas | micrococcus.spp | 21 |
| dibas | neisseria.gonorrhoeae | 23 |
| dibas | porfyromonas.gingivalis | 23 |
| dibas | propionibacterium.acnes | 23 |
| dibas | proteus | 20 |
| dibas | pseudomonas.aeruginosa | 20 |
| dibas | staphylococcus.aureus | 20 |
| dibas | staphylococcus.epidermidis | 20 |
| dibas | staphylococcus.saprophiticus | 20 |
| dibas | streptococcus.agalactiae | 20 |
| dibas | veionella | 22 |
| multi | aspergillus niger_gram stain | 30 |
| multi | bacillus subtilis_gram stain | 26 |
| multi | candida albicans_gram stain | 30 |
| multi | clostridium sporogenes_gram stain | 29 |
| multi | enterococcus faecalis_gram stain | 27 |
| multi | escherichia coli_gram stain | 30 |
| multi | klebsiella pneumoniae_gram stain | 30 |
| multi | pseudomonas auergosa_gram stain | 27 |
| multi | staph aureus_gram stain | 52 |
| multi | streptococcus pyogenes_gram stain | 41 |
