---
title: "Как я искал биткойны в картинах"
date: 2018-03-25
lastmod: 2018-03-25T16:01:23+03:00
draft: false
tags: ["art", "ctf", "bitcoin", "crypto"]
categories: ["blog"]

# you can close something for this content if you open it in config.toml.
comment: false
toc: false
# reward: false
mathjax: false
copyright: false

# you can define another contentCopyright. e.g. contentCopyright: "This is an another copyright."
# contentCopyright: false
---

Совсем недавно наткнулся в одном из телеграмм каналов на новость, о том, что один художник по имени Andy Bauch спрятал в своих картинах ключ к биткойнам.

Меня привлекают переплетения искусства и технологии, поэтому я заинтересовался и решил попробовать свои силы.

Произведения Энди представляют собой рисунки созданные из разноцветных частичек конструктора лего.

Вот одна из работ, в которой спрятаны $20 по курсу на апрель 2016 года:

{{< instagram BHfCvBagaiQ hidecaption >}}

Невооруженным взглядом заметен повторяющийся шаблон. Остается разобраться что он значит.

С помощью gimp сделал картинку понятнее для анализа:

![Gimp processed pattern](/img/bauch/20_BITCOIN_SM.jpg)

Итак, разработаем небольшой скрипт из подручных средств. А под рукой у меня оказался старый добрый PHP и Imagick =) Цель скрипта - получить последовательность бит в соответствии с цветами пикселей картины.

```php
$originFile = storage_path("20_BITCOIN_SM.jpg");
$origin = new \Imagick($originFile);


$len = $request->get('len', 210);

$total = 48*48;

$rW = $len * 2 + 10;
$rH = round($total / $len) * 2 + 10;

$test = new \Imagick();
$test->newImage($rW, $rH, "black");
$test->setImageFormat("png");

$w = 18.9; // ширина пикселя
$h = 18.9; // высота пикселя

$draw = new \ImagickDraw();
$draw->setStrokeColor(new \ImagickPixel('black'));
$draw->setFillColor(new \ImagickPixel('white'));
for ($y = 0; $y <= 47; $y++) {
    for ($x = 0; $x <= 47; $x++) {
	$destin = $origin->getImageRegion($w, $h, 46 + $w * $x, 50 + $h * $y);
	$destin->scaleimage(1, 1);
	if (!$pixels = $destin->getimagehistogram()) {
	    return null;
	}
	$pixels = reset($pixels);
	if (array_sum($pixels->getcolor()) > 256) {
	    $nx = 1 + ($i % $len);
	    $ny = 1 + floor($i / $len);
	    $draw->point($nx, $ny);
	}
    }
}
$test->drawImage($draw);
return response($test->getImageBlob(), 200, ['Content-type' => 'image/png']);
```

Перебор параметра len, как и ожидалось, дал свои плоды:

![bar pattern](/img/bauch/len-210.png)

Сканер штрих-кодов никак не реагирует на последовательность полосок. Идем дальше.

Теперь вместо отрисовки картинки, превратим белые пиксели в 1, а черные в 0;

```php
echo (array_sum($pixels->getcolor()) > 256) ? 1 : 0;
```

Получаем 210 бит, что, конечно, мало для приватного ключа:

```
101001101101000111001100101010000011110011110111001101110111001011010011010011110001111011011100100110110111001110101001101111110010010110001100110110010011001100110100011000110011101000011100011011000011000100
```

Нахожу [делители числа 210](http://www.wolframalpha.com/input/?i=divisors+210): 2, 3, 5, 6, 7, 10, 14, 15, 21, 30, 35, 42, 70, 105

210 на 8 не делиться, значит это не набор байт. Представить в виде изображения матрицу 14x15. На QR код не похоже. А вот на матрице 7x30 проглядывается закономерность - слева больше черных пикселей. Это похоже на битовые коды символов в диапазоне 0-127.

![key matrix](/img/bauch/matrix-7x30.png)

А что если перед каждой группой байт подставить 0?

```php
$bits = "101001101101000111001100101010000011110011110111001101110111001011010011010011110001111011011100100110110111001110101001101111110010010110001100110110010011001100110100011000110011101000011100011011000011000100";
echo "0" . join(" 0", str_split($bits, 7));
```
Получили:

```
01010011 00110100 00111001 01001010 01000001 01110011 01101110 00110111 00111001 00110100 01101001 01110001 01110110 01110010 00110110 01110011 01010100 01101111 01100100 01011000 01100110 01100100 01100110 00110100 00110001 01001110 01000011 01000110 01100001 01000100
```

А теперь этот набор байт превратим в текст [удобным инструментом](https://brainwalletx.github.io/#converter). Вуаля! Получили вполне адекватную строку:

*S49JAsn794iqvr6sTodXfdf41NCFaD*

Очевидно, это не приватный ключ, и вполне возможно что это [Mini private key](https://en.bitcoin.it/wiki/Mini_private_key_format). Что легко проверяется генератором ключей.

Получаю такой адрес:

*1NmxAV1ze28U4Uuqg2fH1JTB8NtWKvTyhM*

Проверяю его баланс и огорчаюсь, что я опоздал на пол дня и кто-то разгадал загадку вперед меня и перевел себе 0.04678122 BTC, что почти 400 баксов по текущему курсу.

https://blockchain.info/address/1NmxAV1ze28U4Uuqg2fH1JTB8NtWKvTyhM

Баланс на Bitcoin Cash по этому адресу так же обращен в 0 :(

Ну что ж, польза от решения загадки осталась - я нашел интересный, полезный и удобный инструмент: [The Cyber Swiss Army Knife](https://gchq.github.io/CyberChef/)

Это была бессонная ночь =)