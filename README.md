# Image-Inpainting-DL-project

## Введение:
В связи с переходом большинства университетов на дистанционное обучение, возросла потребность в записи видеолекций. Но так как преподаватели не имеют должного оборудования для удобной подачи материала, возникает проблема в восприятии контента. Целью нашей работы является удаление объекта с видео (в нашем случае преподавателя). Это поможет избавить видео от посторонних объектов и сфокусировать всё внимание студентов на изучаемом материале.

 1. Выделение человека
На данном этапе производится выделение человека (сегментация). Флагом определяем, какая модель будет при этом работать (fcn или deeplabv3). Описание используемых моделей для сегментации человека:
#### FCN
![FCN](https://github.com/temp-rw/Image-Inpainting-DL-project/blob/master/Images/FCN.png)

**Полностью сверточные сети** (FCN) обязаны своим именем своей архитектуре, которая построена только из локально связанных уровней, таких как свертка, объединение в пул и повышающая дискретизация. Обратите внимание, что в этом типе архитектуры не используется плотный слой. Это уменьшает количество параметров и время вычислений. Кроме того, сеть может работать независимо от исходного размера изображения, не требуя какого-либо фиксированного количества устройств на любом этапе, при условии, что все подключения являются локальными. Для получения карты сегментации (выходной) сети сегментации обычно состоят из 2 частей:
 
 - Путь понижающей дискретизации: захват семантической / контекстной информации.
 - Путь с повышением частоты: восстановить пространственную информацию.

Путь **понижающей дискретизации** используется для извлечения и интерпретации контекста ( *что* ), в то время **как путь повышающей дискретизации** используется для обеспечения точной локализации ( *где* ). Кроме того, чтобы полностью восстановить детализированную пространственную информацию, потерянную в слоях пула или понижающей дискретизации, мы часто используем пропускаемые соединения.
Пропускное соединение - это соединение, которое обходит хотя бы один слой. Здесь он часто используется для передачи локальной информации путем объединения или суммирования карт объектов с пути понижающей дискретизации с картами объектов с пути повышенной дискретизации. Объединение объектов с различными уровнями разрешения помогает объединить контекстную информацию с пространственной информацией.

#### Deeplabv3
-   a) : с Atrous Spatial Pyramid Pooling (ASPP), способным кодировать многомасштабную контекстную информацию.
    
-   b) : с Encoder-Decoder Architecture информация о местоположении / пространстве восстанавливается. Архитектура кодировщика-декодера оказалась полезной в литературе, такой как FPN , DSSD, TDM, SharpMask , RED-Net и U-Net для различных целей.
    
-   c) DeepLabv3 + использует (a) и (b).
    
Кроме того, с использованием Modified Aligned Xception и Atrous Separable Convolution создается более быстрая и прочная сеть.\
![deeplabv3](https://github.com/temp-rw/Image-Inpainting-DL-project/blob/master/Images/Deeplabv3.png)

**DeepLabv3 как кодировщик**

-   Для задачи классификации изображений пространственное разрешение окончательных карт характеристик обычно в 32 раза меньше, чем разрешение входного изображения, и, следовательно, выходной шаг = 32.

-   Для задачи семантической сегментации она слишком мала.

- Можно выбрать выходной шаг = 16 (или 8) для более плотного выделения признаков, удалив шаг в последнем (или двух) блоке (ах) и применив сверточное свечение соответственно.

-  Кроме того, DeepLabv3 дополняет модуль Atrous Spatial Pyramid Pooling , который исследует сверточные объекты в разных масштабах, применяя сверточную свертку с разными скоростями, с помощью функций уровня изображения.

**Предлагаемый декодер**

-   Функции датчика сначала билинейно дискретизируются с коэффициентом 4, а затем объединяются с соответствующими функциями низкого уровня .
    
-   Имеется свертка 1 × 1 для низкоуровневых элементов перед объединением, чтобы уменьшить количество каналов, поскольку соответствующие низкоуровневые элементы обычно содержат большое количество каналов (например, 256 или 512), что может перевесить важность расширенного Особенности кодировщика.
    
-   После конкатенации мы применяем несколько сверток 3 × 3 для уточнения функций, после чего следует еще одно простое билинейное повышение дискретизации с коэффициентом 4.
    
-   Это намного лучше, если сравнивать одно билинейно с дискретизацией 16 × напрямую.


### 2. Расширение изображения

На данном этапе производится расширение изображения. Мы можем проводить уменьшение fps, поскольку соседние кадры похожи, видео относительно статическое. Итоговое значение fps - 5 кадров в секунду.


### 3. Расширение маски
Далее расширяем маску. Поскольку сегментация неточная, нам необходимо добавить в маску выходящие за контур пиксели. Результат расширения маски представлен на картинке:

![Frame](https://github.com/temp-rw/Image-Inpainting-DL-project/blob/master/Images/Frame.png) 

![Mask](https://github.com/temp-rw/Image-Inpainting-DL-project/blob/master/Images/Frame-Mask.png)


### 4. Модель для восстановления изображения

Оптический поток - это движение объектов между последовательными кадрами последовательности, вызванное относительным движением между объектом и камерой. Проблема оптического потока может быть выражена как:

![Optical-Flow](https://github.com/temp-rw/Image-Inpainting-DL-project/blob/master/Images/Optical_flow.png)

где между последовательными кадрами мы можем выразить интенсивность изображения (I) как функция пространства (x,y) и время (t). Другими словами, если мы возьмем первое изображение I(x,y,t) и переместим его пиксели (dx, dy) над t, мы получаем новое изображение I(x+dx,y+dy,t+dt).
Во-первых, мы предполагаем, что интенсивность пикселей объекта постоянна между последовательными кадрами. Во-вторых, мы берем приближение RHS для ряда Тейлора и удаляем общие термины. В-третьих, мы делим на dt для вывода уравнения оптического потока. 
Первая подсеть оценивает поток в относительно грубом масштабе и подает их во вторую и третью подсети для дальнейшего уточнения. На втором этапе, после того, как поток получен, большинство пропущенных областей может быть заполнено пикселями в известных областях посредством потока распространения из разных кадров.

![Left: Sparse Optical Flow ; Right: Dense Optical Flow](https://github.com/temp-rw/Image-Inpainting-DL-project/blob/master/Images/sparse-vs-dense.gif)

Сначала вычисляем оптический поток для всего изображения, используя архитектуру FlowNet 2.0.

На первом этапе предлагается сеть глубокого завершения Deep Flow Completion Network (DFC-Net). DFC-Net состоит из трех аналогичных подсетей, названных DFC-S. Первая подсеть оценивает поток в относительно грубом масштабе и подает их во вторую и третью подсети для дальнейшего уточнения. На втором этапе, после получения потока, большинство пропущенных областей могут быть заполнены пикселями в известных областях посредством распространения потока из разных кадров. Наконец, используется обычная сеть с окраской изображений для заполнения оставшихся областей, которые не видны во всем видео. Благодаря высокому качественному расчетному потоку на первом этапе мы можем легко распространить эти результаты покраски изображения на всю видеопоследовательность.

Во время обучения для каждого видеоряда, произвольно генерируйте недостающие области. Оптимизация цель состоит в том, чтобы минимизировать расстояние между предсказаниями и истинным значением. Сначала предварительно обучаются три подсети по отдельности, а затем совместно отлаживаются.

![DFC1](https://github.com/temp-rw/Image-Inpainting-DL-project/blob/master/Images/DFC1.png)

DFC-S использует ResNet-50 в качестве основы.  ResNet-50 состоит из пяти блоков, названных «conv1», «conv2 x» - «conv5 x».  Мы модифицируем входной канал первой свертки в «conv1», чтобы соответствовать форме наших входов (например, 33 в первом DFC-S).  Чтобы увеличить разрешение, мы уменьшаем сверточные шаги и заменяем извилины расширенными извилинами от «conv4 x» до «conv5 x», аналогично.  Модуль повышения дискретизации, который состоит из трех чередующихся сверток, реляционных и повышающих слоев, добавляется для увеличения прогноза.  Чтобы спроецировать прогноз на поле потока, мы удалили функцию активации по активации в модуле отбора проб.

![DFC2](https://github.com/temp-rw/Image-Inpainting-DL-project/blob/master/Images/DFC2.png)

Затем то, что под маской, восстанавливаем, используя предыдущий или следующий кадр. И исходя из полученного, совмещаем пиксели. Здесь модели не дорисовывает, а именно совмещает пиксели. Модель учится совмещать существующие пиксели со скрытыми, причём процесс совмещения идёт итерационно. За один подход предсказываются не все пиксели, поэтому полное совмещение достигается за определенное количество итераций.

В итоге на обработку видео длительностью 50 секунд (5 кадров в секунду) было затрачено 2 часа.
