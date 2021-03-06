На стеке был задан [вопрос](https://ru.stackoverflow.com/questions/649761/%D0%9C%D0%BE%D0%B6%D0%BD%D0%BE-%D0%BB%D0%B8-%D0%BF%D1%80%D0%B5%D0%B4%D1%81%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D1%8C-%D0%BC%D0%B0%D0%BA%D1%81%D0%B8%D0%BC%D0%B0%D0%BB%D1%8C%D0%BD%D1%8B%D0%B9-%D1%80%D0%B0%D0%B7%D0%BC%D0%B5%D1%80-png-%D0%B8%D0%B7%D0%BE%D0%B1%D1%80%D0%B0%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F/651646#651646):

Я делаю клипборд, и хотел бы оптимизировать выделение памяти, а точнее исключить лишнюю реаллокацию. Допустим, я хочу сохранить изображение как png файл, существует ли какой-нибудь способ узнать максимальный размер будущего файла? Конечно меня интересуют какие нибудь оптимальные результаты (например мы можем оценить сверху размером битмапы - только толку от этого мало)

# Ответ #

Вы задали сложный и интересный вопрос, точного ответа на который скорее всего ни у кого нет. Хотя Вы можете поискать какие-нибудь оценки алгоритма сжатия `DEFLATE`, который используется в `PNG`. К слову, о `PNG`. Это формат хранения картинок со сжатием данных без потерь, в котором используется алгоритм [DEFLATE][1]. Метод не из самых простых. Его достаточно хорошее описание мной найдено было найдено [здесь][2], [здесь][3], частично [тут][4] и [вот тут][5]. Но в силу того, что мои интересы лежат немного в стороне от сжатия данных, вспоминать коды Хаффмана в различных вариациях я не стал, а избрал иной, более дешёвый подход к решению задачи. Он даёт худший результат и не точный результат. Но, при должном развитии идеи, которую я опишу, можно получить достаточно качественные нижнюю и верхнюю оценки. Нужно также понимать, что используюя данный подход, всегда можно будет подобрать такое изображению, которое будет выбиваться из общей динамики картинок. В силу того, что я не знаю, что и как Вы хотите оптимизировать, приведу лишь основные положения того, что можно делать. Всё ниже перечисленное снабжу кодом на `python 2.7`. Готового решения, к сожалению, я не дам, посколько не знаю целей. Но на основе моих наработок можно получить очень хороший результат.

Насчёт точной оценки. Её скорее всего можно получить, изучив то, как ведут себя коды Хаффмана. Но это будет трудоёмкий процесс. Вопрос в том, насколько это Вам нужно. 

Общая идея подхода состоит в том, чтобы выявить зависимость 

[![Зависимость][6]][6], где

[![введите сюда описание изображения][7]][7] -- параметры, которые будем задавать. Указанную зависимость мы будем получать, рассматривая некоторое множество изображений (у меня на компьютере есть база со стоковыми фотографиями профессиональных фотографов). Эти изображения мы будем рассматривать как объекты. Выше указанную зависимость мы будем строить, основываясь на выборке картинок.

Если у Вас нет `python` и `IDE` для него, то я советую Вам `pyCharm`. Все вопросы по установке, либо возникающим проблемам Вы можете задавать в комментариях. Начиная работать, обязательно обратите внимание на pip -- установщик пакетов, который, кстати, очень хорошо интегрирован в `pyCharm`.

Теперь насчёт изображений. В сети полно всяких дампов фоток, картинок и другого хлама. Вы легко их можете найти в гугле. Но проблема в том, что они могут быть в разных форматах. Эта проблема легко решает приведенеием их к `*.png`. Сделать это можно так:

    import Image
    import glob
    
    
    listOfImages = glob.glob("/home/hedgehogues/project/testPNG/*.jpg") # Получаем список имён файлов
    
    index = 0
    for itemFile in listOfImages:
        img = Image.open(itemFile)
        img.save("/home/hedgehogues/project/testPNG/" + str(index) + ".png")
        index += 1
        print index

Код комментить не буду. Он вроде бы понятен.

Итак, начал я с того, что решил поглядеть, что представляет собой размер картинки (далее картинка -- это некоторый экземпл базы картинок) в зависимости от её площади. К сожалению, здесь меня ждал грустный ответ. Это не очень хорошое распределение. Вы можете на него взглянуть:

[![Зависимость площади от размеров картинки в битах][8]][8]

Но вообще говоря, даже глядя на такую картинку можно дать кое-какие оценки. Например, мы можем дать жёсткую оценку вёрхней границы. Она, как видно, из рисунка выражается следующим уравнением:

[![Верхняя граница][9]][9]

Нижняя граница соответственно:

[![Нижняя граница][10]][10]

Таким образом, размерах файлов при фиксированной площади может сильно варьироваться (практически в 3 раза) ~ 2.922. Для того, чтобы понять, что это много, можно сравнить 3 Мб и 9 Мб. 300 кБ и 1Мб. Разница ощутима. Апогея она достигнет, если рассматривать изображения очень больших размеров: 50 Мб и 150 Мб. Каково?

Я приведу код, с помощью которого можно произвести рассчёты:

    import Image
    import os
    import matplotlib.pyplot as plt
    import numpy as np
    
    
    listOfSize = [] # Размер файла
    total = [] # Площадь картинки
    
    
    index = 0
    for itemFile in range(0, 260):
        tmpSize = os.path.getsize("/home/hedgehogues/project/testPNG/" + str(index) + ".png") # Получаем размеры файла с картинкой
        img = Image.open("/home/hedgehogues/project/testPNG/" + str(index) + ".png")
        hLocal, wLocal = img.size # Размеры сторон картинки
        img = np.array(img.convert('L'))
        listOfSize.append(tmpSize)
        total.append(wLocal * hLocal)
        index += 1
        print index
    
    plt.plot(total, listOfSize, linestyle = '', marker = 'x')
    plt.show()

Следующим логичным шагом стало предположение о том, что, информация о длинах сторон картинок -- это совсем неинформативный признак, поскольку внутри каждой картинки кроется различная информация, а следовательно, самые информативные параметры будут связаны с интенсивностями пикселей. Здесь я оговорюсь, что далее, для простоты, все изображения я буду приводить к градациям серого. Разумеется, если этого не делать, а работать со всей информацией, то мы получим более качественные результаты. Но и сложность задачи возрастает в разы.

Для учёта этой информации будем строить [гистограммы][11] изображений:

[![Гистограмма][12]][12]

Но, вот незадача, мои изображения оказываются очень большими (5000х5000) и их обработка занимает порядочное время. Поэтому я их жму. Получаю:

[![Сжатое изображение][13]][13]

Как видим, характер гистограммы сохраняется. Приведу код, который позволяет построить такого класса гистограммы:

    import Image
    import matplotlib.pyplot as plt
    import numpy as np


    index = 0
    for itemFile in range(0, 250):
        img = Image.open("/home/hedgehogues/project/testPNG/" + str(index) + ".png")
        img.thumbnail((300, 300), Image.ANTIALIAS) # сжатие изображения
        img = np.array(img.convert('L'))
        y = np.histogram(img, bins = range(0, 256)) # bins задачёт количество градаций гистограммы
        x = np.arange(0, 1, 1./255)
        plt.plot(x, y[0])
        plt.show()
        index += 1
    
    
    
        print index


  Что со всем этим делать? Легко. Можно построить аппроксимацию этих гистограмм. Это можно делать по-разному. Например, при помощи нелинейного [МНК][14]. Я подобрал такую функцию, которая более или менее отвечает гистограммам. Все гистограммы для МНК нормируются по отрезку `[0; 1]`. Вот, что мы получаем. Несколько гистограмм и построенных для них МНК:

[![Гистограмма 1][15]][15]
[![Гистограмма 2][16]][16]
[![Гистограмма 3][17]][17]
[![Гистограмма 4][18]][18]
[![Гистограмма 5][19]][19]

Данная функция подбиралась методом научного тыка и выглядит следующим образом:

[![Целевая функция][20]][20]

Средствами python легко найдём неизвестные параметры, если минимизировать будем функцию, которая соответствует МНК:

[![Функция минимизации][21]][21]

Для минимизации перебёрем все данные для каждой гистограммы. Все эти действия можно проделать самостоятельно:

    import Image
    import scipy.optimize as opt
    import matplotlib.pyplot as plt
    import numpy as np
    
    
    # Целевая функция
    def Model(a, x):
        sum = a[0]
        for coeff in range(1, len(a)):
            sum += a[coeff] * ((x * np.sin(x)) ** coeff + np.exp(x))
        return sum
    
    
    
    
    index = 0
    for itemFile in range(0, 250):
        img = Image.open("/home/hedgehogues/project/testPNG/" + str(index) + ".png")
        img.thumbnail((300, 300), Image.ANTIALIAS)
        img = np.array(img.convert('L'))
        weight = np.array(range(0, 10))
        ErrorFunc = lambda tpl, x, y: 0.5 * (Model(tpl, x) - y) ** 2 # Функционал минимизации
        y = np.histogram(img, bins = range(0, 256))
        x = np.arange(0, 1, 1./255) # Нормировка
        y = y[0] / float(np.max(y[0])) # Нормировка
        spl = opt.leastsq(ErrorFunc, weight, args = (x, y)) # Вычисление коэффициентов
        yy = Model(spl[0], x)
        plt.plot(x, yy)
        plt.plot(x, y)
        plt.show()
        index += 1
    
    
    
        print index

Получим веса `w_i`, а также имея площадь, можем построить ещё одно регрессионную модель, которая будет предсказывать размер конкртеного изображения. Сделать это можно по аналогии с тем, как построена регриссионная модель выше. Введём некоторую целевую функцию и функционал минимизации. Запишем исходные данные в виде: `u_i = [w_i, area]`. Теперь имея в качестве исходных данных пары `(u_i, total_size)`, аналогичным образом обучим модель и получим некоторую зависимость. По указанной зависимости можно будет предсказывать предполагаемый размер файла.

С другой стороны, можно воспользоваться более простой идеей и также, как и ранее, получить верхнюю и нижнюю оценку. Для этого посчитаем среднее значение элементов гистограммы. Построим график зависимости размера файла от среднего значения:

[![График зависимости размера файла от среднего значения][22]][22]

Предвосхищая вопросы. Замечу, что на графике присутствуют два вида точек. Синие -- это множество, на котором производилось "обучение". Красные -- это точки, взятые из интернетов (картинки скачал). Как видим, они примерно укладываются в общую тенденцию. Разумеется, в данной ситуации у нас есть выбросы, которые нужно отдельно обработать и понять их причину. Кроме того, наша оценка средним значением гистограммы очень груба. А значит не следует претендовать на слишком качествеенный результат. Также отмечу, что построение гистограммы для отдельного изображения -- это затратная операция. Поэтому имеет смысл брать некотору аппроксимацию этой операции (например, брать на изображении случайные пиксели и строить гистограмму по ним). 

Приведу код:

    import Image
    import os
    import matplotlib.pyplot as plt
    import numpy as np
    
    
    
    def Model(a, x):
        sum = a[0]
        for coeff in range(1, len(a)):
            sum += a[coeff] * ((x * np.sin(x)) ** coeff + np.exp(x))
        return sum
    
    listOfSize = []
    listOfSizeTest = []
    h = []
    w = []
    total = []
    totalTest = []
    data = []
    
    
    # Перебираем все элементы из train set
    index = 0
    for itemFile in range(0, 250):
        img = Image.open("/home/hedgehogues/project/testPNG/" + str(index) + ".png")
        img.thumbnail((300, 300), Image.ANTIALIAS)
        img.save("/home/hedgehogues/project/testPNG/_-1.png")
        tmpSize = os.path.getsize("/home/hedgehogues/project/testPNG/_-1.png")
        img = np.array(img.convert('L'))
        y = np.histogram(img, bins = range(0, 256))
        total.append(np.mean(y[0] / float(np.max(y[0]))))
        index += 1
    
    
    
        print index
    
    # Перебираем все элементы из test set (картинки из интернетов)
    for itemFile in range(0, 6):
    
        img = Image.open("/home/hedgehogues/project/testPNG/_" + str(itemFile) + ".png")
        img.thumbnail((300, 300), Image.ANTIALIAS)
        img.save("/home/hedgehogues/project/testPNG/_-1.png")
        tmpSize = os.path.getsize("/home/hedgehogues/project/testPNG/_-1.png")
        img = np.array(img.convert('L'))
        listOfSizeTest.append(tmpSize)
        y = np.histogram(img, bins = range(0, 256))
        totalTest.append(np.mean(y[0] / float(np.max(y[0]))))
    
    
    plt.plot(total, listOfSize, linestyle = '', marker = 'x')
    plt.plot(totalTest, listOfSizeTest, linestyle = '', marker = 'x', color = 'red')
    plt.show()


  [1]: https://ru.wikipedia.org/wiki/Deflate
  [2]: http://fat-crocodile.livejournal.com/194295.html
  [3]: http://compression.ru/download/articles/lz/mihalchik_deflate_decoding.html
  [4]: http://fat-crocodile.livejournal.com/191957.html
  [5]: https://habrahabr.ru/post/274825/
  [6]: https://i.stack.imgur.com/GMpUY.gif
  [7]: https://i.stack.imgur.com/rrYZp.gif
  [8]: https://i.stack.imgur.com/wnW75.png
  [9]: https://i.stack.imgur.com/XyTDl.gif
  [10]: https://i.stack.imgur.com/daFOO.gif
  [11]: https://ru.wikipedia.org/wiki/%D0%93%D0%B8%D1%81%D1%82%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B0
  [12]: https://i.stack.imgur.com/M4dnc.png
  [13]: https://i.stack.imgur.com/BiIgi.png
  [14]: https://ru.wikipedia.org/wiki/%D0%9C%D0%B5%D1%82%D0%BE%D0%B4_%D0%BD%D0%B0%D0%B8%D0%BC%D0%B5%D0%BD%D1%8C%D1%88%D0%B8%D1%85_%D0%BA%D0%B2%D0%B0%D0%B4%D1%80%D0%B0%D1%82%D0%BE%D0%B2
  [15]: https://i.stack.imgur.com/gG5I3.png
  [16]: https://i.stack.imgur.com/hLZ1S.png
  [17]: https://i.stack.imgur.com/GgIsf.png
  [18]: https://i.stack.imgur.com/aPn6E.png
  [19]: https://i.stack.imgur.com/5sL8h.png
  [20]: https://i.stack.imgur.com/oWoMb.gif
  [21]: https://i.stack.imgur.com/HK0N5.gif
  [22]: https://i.stack.imgur.com/q1Z2b.png