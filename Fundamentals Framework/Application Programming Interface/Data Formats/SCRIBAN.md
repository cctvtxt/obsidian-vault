У LiquidJS есть встроенный механизм работы с файловой системой. Вы один раз указываете папку, и дальше просто вызываете файлы по имени.


```javascript
const { Liquid } = require('liquidjs');

const engine = new Liquid({
    root: './templates', 
    extname: '.liquid' 
});

async function main() {
    const result = await engine.renderFile('greeting', {
        name: "Алексей",
        orderId: 777,
        isPremium: true
    });

    console.log(result);
}

main();
```



```scriban
Привет, {{ name }}!
Ваш заказ №{{ orderId }} успешно оформлен.

{% if isPremium %}
Спасибо, что используете Premium подписку!
{% endif %}
```