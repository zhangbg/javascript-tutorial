1. Да, верно, с оговоркой. Элемент `elem.lastChild` последний, у него нет правого соседа.

    **Оговорка:** `elem.lastChild.nextSibling` выдаст ошибку если `elem` не имеет детей.

2. Нет, неверно, это может быть текстовый узел. Значением `elem.children[0]` является первый узел-элемент, перед ним может быть текст.

    Аналогично предыдущему случаю, если у `elem` нет детей-элементов -- будет ошибка.