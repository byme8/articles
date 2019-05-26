Время от времени, когда я читал о Roslyn и его анализаторах, у меня постоянно возникала мысль: "А ведь этой штукой можно сделать nuget, который будет ходить по коду и делать кодогенерацию". Быстрый поиск не показал ничего интересного, по этому было принято решение копать. Как же я был приятно удивлен, когда обнаружил что моя затея не только реализуемая, но все это будет работать почти без костылей. 

И так кому интересно посмотреть на то как можно сделать "маленькую рефлексию" и запаковать ее в nuget прошу подкат.

 <cut/>

# Введение

 Думаю первое что стоить уточнить это то, что понимается под "маленькой рефлексией". Я предлагаю реализовать для интерисующих нас типов метод ``` Dictionary<string, Type> GetBakedType() ``` который будет возвращать имена пропертей и их тип. 

Ручная его реализация будет иметь следующий вид:
 ``` cs

using System;
using System.Collections.Generic;

public static class testSimpleReflectionUserExtentions
{
    private static Dictionary<string, Type> properties = new Dictionary<string, Type>
        {
            { "Id", typeof(System.Guid)},
            { "FirstName", typeof(string)},
            { "LastName", typeof(string)},
            { "Parent", typeof(testSimpleReflection.User)},
        };

    public static Dictionary<string, Type> GetBakedType(this global::testSimpleReflection.User value)
    {
        return properties;
    }
}

 ```

 Ничего сверхъестественного здесь нет, но реализовывать его для для всех типов это муторное и не интересное дело. Коме того грозит нам обязательными опечатками. Но мы ведь программисты так ведь? Почему бы не попросить компилятор нам помочь.

 Вот тут на арену выходит Roslyn с его анализаторами. Они предоставляют возможность проанализировать код, а потом и изменить его. Так давайте же научим компилятор новому трюку. Пускай он ходит по коду, смотрит для каких типов мы еще не реализовали наш ``` GetBakedType ``` и реализовывает его.

 # Предварительная подготовка

Перед тем как начнем необходимо установить компонент 'Visual Studio extention development' в студийном Installer-е. Дополнительно для VS 2019 нужно не забыть выбрать ".NET Compiler Platform SDK" с правой стороны в опциональных компонентах.

Теперь мы можем запустить студию и создать проект "Analyzer with Code Fix". Первым делом предлагаю удалить пример анализатора чтобы он нам не мешался. Вот теперь мы готовы начинать.






