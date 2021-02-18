---
layout: post
title: LCS algoritmus
---

> Jelen leírást elsősorban a BME 4. féléves Algoritmuselmélet tárgyának hallgatói számára készítettem.  
> Természetesen mások számára is hasznos lehet, de a cikk írásánál azzal az előfeltételezéssel éltem, hogy a fő célközönség már elvégezte a BME mérnökinformatikus szak első 3 félévét.

A legtöbb programozással foglalkozó szakember, vagy az éppen csak programozni tanuló érdeklődők már találkoztak olyan programmal, ami két programokód közötti különbséget generál.

Ezt a különbséget angolul diff-ként szokás emlegetni, ami sokat segíthet, ha például egy szöveg két változata közti különbségeket akarjuk vizualizálni.

![GitHub diff](/images/articles/lcs_algoritmus/01_diff_github.png)

A képen egy nyílt forráskódú C++ könyvtár, a [cpp-peglib](https://github.com/yhirose/cpp-peglib) két különböző állapota közti különbség látható. A bal oldali állapot az időben korábbi, a jobb olali pedig a későbbi állapotát mutatja a könyvtárnak. Ebben a konkrét esetben a C++ 2017-es szabványában megjelent `string_view` osztályt kezdték el használni a könyvtár fejlesztői. A konkrét változtatás, amiről a képet készítettem, [itt](https://github.com/yhirose/cpp-peglib/commit/217d5517443e6c082bf682aae1557933f885a1bb#diff-5ca9fbaaa7735ba1e069e166f23ab9553fa99963defa0cd7762b4604e8f52c5dR3955) érhető el.

Ahhoz, hogy a bal oldali állapotból a jobb oldali állapotba jussunk, a pirossal kiemelt sorokat előbb törölnünk kell, majd a jobb oldalon zölddel kiemelt sorokat hozzáadni a helyükre - ezt a diff-et a GitHub az LCS, avagy Longest Common Subsequence (Leghosszabb Közös Részsorozat) algoritmus segítségével állította elő.

Ebben a cikkben ezt a matematikai problémát, az LCS problémát fogom bemutatni, a megoldására két algoritmust bemutatva: rekurzív függvényhívásokkal, illetve dinamikus programozással. Mindkettő algoritmust C++ és C# nyelveken mutatom majd be, a teljes kód egy külön GitHub repositoryban elérhető.

Meg kell jegyezni, hogy a dinamikus programozás tulajdonképpen önmaga is egy rekurzív megoldási módszer, csak rekurzív függvényhívások helyett a korábbi eredményeket eltároljuk.

## Az LCS probléma

Mielőtt az LCS problémát leírnánk formálisan, tekintsünk át néhány hozzá kapcsolódó fogalmat, és nézzünk meg pár egyszerű példát! Először az LCS rövidítés egyes szavainak jelentésével fogunk foglalkozni, utána pedig pontosan is definiáljuk magát a problémát. Nézzük a Leghosszabb Közös Részsorozat kifejezést hátulról!

### Részsorozat?

Először is, mit jelent a részsorozat ebben a kifejezésben?

Vegyünk egy listát, és jelöljük a listát `L`-lel (a lista bármilyen elemeket tartalmazhat, lehet szó karaktekről, teljes szavakról, mondatokról, számokról vagy bármi egyéb adatról).

Ekkor ennek az `L` listának `L'` részsorozata, hogyha az alábbi két feltétel teljesül:
  - `L'` minden eleme egyúttal eleme az eredeti `L` listának is
  - `L'` elemei az eredeti `L` listában olyan sorrendben követik egymást, mint ahogy `L'`-ben

Fontos megjegyezni, hogy az `L'` elemeinek nem kell feltétlenül közvetlenül egymás mellett állnia az eredeti `L` listában.

Vegyünk egy egyszerű példát: legyen `L = ['A', 'B', 'C', 'D']`. Soroljuk fel most `L` minden részsorozatát:

`L` 1 elemű részsorozatai:
  - `['A']`
  - `['B']`
  - `['C']`

`L` 2 elemű részsorozatai:
  - `['A', 'B']`
  - `['A', 'C']` (!)
  - `['A', 'D']` (!)
  - `['B', 'C']`
  - `['B', 'D']` (!)
  - `['C', 'D']`

`L` 3 elemű részsorozatai:
  - `['A', 'B', 'C']`
  - `['A', 'B', 'D']` (!)
  - `['A', 'C', 'D']` (!)
  - `['B', 'C', 'D']`

`L` 4 elemű részsorozatai:
  - `['A', 'B', 'C', 'D']`

(!)-el jelöltem azokat a részsorozatokat, ahol legalább egy helyen van egy "ugrás", avagy a részsorozat két szomszédos eleme nem volt szomszédos az eredeti `L` listában.

> Apró megjegyzés: létezik a részlista fogalma is. A részlisták esetében megköveteljük, hogy az eredetileg szomszédos elemek továbbra is szomszédosak maradjanak.  
> Ennek megfelelően a (!)-el jelölt listák nem lennének az eredeti `L` lista részlistái, ők csak részsorozatokat alkotnak `L`-ben.  
> Ebben a cikkben a részlistákkal többet nem fogunk foglalkozni, azok egészen más feladatok megoldására hasznlhatóak.

### Közös részsorozat?

Ha már tudjuk azt, hogy mi az a részsorozat, nem nehéz megérteni a közös részsorozat fogalmát se:

Vegyünk kettő listát, `L1`-et és `L2`-őt. Ekkor egy harmadik `L'` lista közös részsorozata `L1`-nek és `L2`-nek, ha mindkettejüknek részsorozata.

Legyen például `L1 = ['A', 'B', 'C']` és `L2 = ['B', 'C', 'D']`.

Ekkor az alábbi részsorozatok találhatók meg mindkettő listában:
  - `['B']`
  - `['C']`
  - `['B', 'C']`

### Leghosszabb közös részsorozat?

A fenti fogalmak birtokában a leghosszabb közös részsorozatot definiálni már igazán könnyű.

Vegyünk kettő listát, `L1`-et és `L2`-őt. Ekkor a két lista leghosszabb közös részsorozata az összes közös részsorozatuk közül a leghosszabb.

Legyen például megint `L1 = ['A', 'B', 'C']` és `L2 = ['B', 'C', 'D']`.

Azt már fent láttuk, hogy ennek a két listának 3 közös részsorozata van, ezek közül a leghosszabb: `L' = ['B', 'C']`.

## Az LCS probléma (még egyszer)

Most, hogy a szükséges fogalmakat már átnéztük, definiálhatjuk magát az LCS problémát. Az előző fogalmak ismeretében ez már egészen egyszerű:

  - A probléma bemenete: kettő lista, `L1` és `L2`
  - A probléma kimenete: `L1` és `L2` leghosszabb közös részsorozata

Persze felmerülhet kérdésként, hogy ha ismerjük két lista leghosszabb közös részsorozatát, abból hogyan tudunk egy olyan "diff" nézetet generálni, mint amit a fenti képen lehet látni. Ez egy picit nehezebb lesz, de egy olyan algoritmust fogunk tervezni, ami ezt a problémát is megoldja.

## A dinamikus programozásról általában

Mint már írtam, az LCS probléma egy lehetséges algoritmikus megoldását dinamikus programozással fogom bemutatni. Érdemes röviden megemlékezni arról, hogy mégis mi az a dinamikus programozás.

A dinamikus programozást a számítástechnika ősi idejében, az 1950-es években egy [Richard Bellman](https://hu.wikipedia.org/wiki/Richard_Bellman) nevű matematikus dolgozta ki. 

A módszer lényege a következő: egy valamilyen matematikai problémára szeretnénk algoritmikus megoldást találni. A célunk az, hogy az eredeti problémát felbontsuk több, önmagához hasonlító kisebb problémára, majd azokat tovább bontsuk rekurzívan, amíg egy triviálisan megoldható problémát kapunk.

Ha valaki emlékszik a korábbi programozás tanulmányaira, akkor felidézhet hasonló algoritmusokat: elsőként a Fibonacci számsorozat, és a gyorsrendezés (quicksort) juthat eszünkbe.

Az előző kettő algoritmusból azonban csak az egyik, a Fibonacci számsor kiszámítása igazi dinamikus programozás: 
  - A Fibonacci számsor kiszámításakor az `n.` elem kiszámításához az `n-1.` és `n-2.` elem értékére van szükségünk. Ez a két részprobléma nehézségben és jellegében is hasonlít az eredeti problémára.
  - A quicksort alkalmazásakor a kezdetben megkapott listát minden lépésben megfelezzük, és a kapott részlistákat rendezzük a quicksort rekurzív meghívásával.  Ez a megoldás bár a quciksortot hívja meg rekurzívan, azonban a kapott új részproblémák jóval egyszerűbbek az eredetinél. Ez az oka annak, hogy nem tekintjük dinamikus programozási algoritmusnak a quicksort-ot.

> Kicsit pontosabban definiálva a fenti érvelést: a dinamikus programozás esetén megköveteljük, hogy a keletkező részproblémák  "átfedőek" legyenek.  
> Ez a Fibonacci számsor esetében teljesül: az `n-2.` elem kiszámítására mind az `n-1.`, mind az `n.` elem kiszámítására szükségünk van.  
> A quciksort esetén ha a tömb első rendezését elvégeztük, majd elkezdjük a résztömböket rendezni, az új részproblémák egymástól teljesen függetlenek. Emiatt ebben az esetben a részproblémák nem "átfedőek".

### Általános technikák dinamikus programozásban

Rengeteg különböző probléma van, amely megoldható dinamikus programozással, ám van néhány hasonló tulajdonságuk, amiket érdemes kiemelni:

  - Megoldhatóak rekurzív függvényhívásokkal, vagy listákkal (tömbökkel).
    - A listák (vagy tömbök) általában a rekurzív függvényhívásokat helyettesítik.
    - Röviden: egy valós számítógépen a tárhelyigény növelésével egy algoritmus futásidejét csökkenthetjük (a köztes számítások eredményének tárolásával).
  - A probléma bementét a legtöbb esetben minden rekurzív hívásnál 1 elemmel rövidítik le.
  - A rekurzió végén a probléma egy olyan állapotba, amely már triviálisn megoldható.

Néhány közismert probléma, ami megoldható dinamikus programozással:

  - Legrövidebb út keresés gráfban - Dijkstra algoritmusa is dinamikus programozást használ, csakúgy, mint a Bellman-Ford algoritmus.
  - Az LCS probléma - nemsokára rátérünk a tényleges algoritmusra.
  - [Glob pattern matching](https://en.wikipedia.org/wiki/Glob_(programming)) - olyan egyszerű szöveges minták, amik `'?'` és `'*'` wildcard karaktereket tartalmazhatnak.
  - [Fibonacci számsor](https://hu.wikipedia.org/wiki/Fibonacci-sz%C3%A1mok)
  - [Hanoi tornyai](https://hu.wikipedia.org/wiki/Hanoi_tornyai)

## Tervezzünk algoritmust!

Eddig megfogalmaztuk az LCS problémát, átnéztük a dinamikus programozás általános ismérveit, és megismertünk néhány ismert dinamikus programozással megoldható problémát. Itt az ideje, hogy megtervezzük a tényleges algoritmust az LCS probléma megoldására!

Először koncentráljunk egy jelentősen egyszerűbb probléma megoldására: határozzu meg két lista leghosszabb közös részsorozatának hosszát! Ez jelentősen le fogja egyszerűsíteni a problémát, és ezt felhasználva magát a leghosszabb közös részsorozatot is meg tudjuk majd határozni.

Fent már említettem, hogy a dinamikus programozás során általában arra törekszünk, hogy a bemenetként kapott lista hosszát szépen lassan, egyesével csökkentsük, míg egy nagyon egyszerű esethez nem jutunk el. Hogyan tudjuk ezt megtenni ebben az esetben?  
A  bemenet jelen esetben kettő listából áll, így nem lehet annyival elintézni a probléma egyszerűsítését, hogy "hagyjuk el a lista utolsó elemét". Egy olyan problémát, aminek a bemenete kettő lista, összesen három féle módon lehetséges egyszerűsíteni:
  1. Az első lista végéről hagyunk el egy elemet
  1. A második lisa végéről hagyunk el egy elemet
  1. Az első, és a második lista végéről is elhagyunk egy-egy elemet.

A rekurzió során minden egyes lépésnél két listával dolgozunk, amik az eredeti listák rövidebb változatai. Egy rekurzív lépés során mindig az aktuális két lista utolsó kettő karakterét hasonlítjuk össze:
  - ha az utolsó két karakter megegyezik, akkor úgy vesszük, hogy az benne van a leghosszabb közös részsorozatban - ekkor mindkettő lista végéről törölnünk kell egy-egy elemet, és a maradék listákra kell a rekurzív algoritmust meghívnunk
  - ha ez a két karakter eltér, akkor meg kell néznünk, hogy mi történik, ha az első lista végéről törlünk egy elemet, és a második listát változatlanul hagyjuk; illetve fordítva. Eközül a kettő lehetőség közül pedig a hosszabbat kell választanunk.

Így tehát a célunk az, hogy ha az előbbb leírt három esetre már tudjuk a választ, akkor az eredeti kérdést is meg tudjuk válaszolni - az egyszerűsített problémákat pedig rekurzívan tovább egyszerűsítjük, amíg elérünk egy triviális esetet.  
A lehetséges egyszerűsítések mellett még meg kell határoznunk a triviális eseteket, amikor megállítjuk a rekurzív problémamegoldást. Általában, ha egy problémának egy lista (vagy esetleg több lista) a bemenete, akkor az üres, vagy egy elemű lista elérésére törekszünk. Így tehát a következő 2 triviális esetet mondhatjuk ki:
  1. Két üres lista leghosszabb közös részsorozatának hossza 0.
  1. Egy üres, és egy nem üres lista leghosszabb lista leghosszabb közös részsorozatának hossza szintén 0.

A következő ábrán egy rövid példán látható az algoritmus futása:
![Example flowchart](/images/articles/lcs_algoritmus/02_flowchart.png)

Ezzel igazából meg is vagyunk az algoritmus megtervezésével, már csak annyi maradt hátra, hogy lekódoljuk magát az algoritmust. Ezt először rekurzív függvényhívásokkal fogjuk elvégezni, majd a rekurzív függvényhívások helyett egy kétdimenziós tömbben tároljuk el a részeredményeket.

Az rekurzív algoritmus tehát a következőképpen fog kinézni, C++ nyelven megírva:
```cpp
template<typename TYPE>
int lcs(std::list<TYPE> const & list1, std::list<TYPE> const & list2)
{
  if(list1.empty())
    return 0;
  if(list2.empty())
    return 0;

  auto list1_shorter = list1;
  auto list2_shorter = list2;
  list1_shorter.pop_back();
  list2_shorter.pop_back();

  if(list1.back() == list2.back())
  {
    return lcs(list1_shorter, list2_shorter);
  }
  else
  {
    int len1 = lcs(list1, list2_shorter);
    int len2 = lcs(list1_shorter, list2);
    return len1 > len2 ? len1 : len2;
  }
}
```

Ez a kód természetesen nem optimális, hiszen a rövidebb listák képzéséhez előbb lemásolja mindkettő listát, majd a `pop_back` tagfüggvény meghívásávl törli ki az utolsó elemet. Ehelyett sok más optimális megoldás elképzelhető, a C++ nyelvhez legjobban illeszkedő megoldás az lenne, ha egy-egy iterátorpárt kapna az `lcs` függvény, ami két konténer első elemét, illetve a végét jelezné.  
Ekkor figyelnünk kell arra, hogy a C++ nyelvben a `.end()` tagfüggvény mindig az utolsó utáni elemre mutat, így hogy az utolsó elemet megkapjuk, egy elemmel hátrébb kell lépnünk.

Az előző kód átírása után ezt kapjuk:
```cpp
template<typename ITERATOR>
int lcs(ITERATOR begin1, ITERATOR end1, ITERATOR begin2, ITERATOR end2)
{
  if(begin1 == end1)
    return 0;
  if(begin2 == end2)
    return 0;

  if(*(end1 - 1) == *(end2 - 1))
  {
    return lcs(begin1, end1 - 1, begin2, end2 - 1);
  }
  else
  {
    int len1 = lcs(begin1, end1, begin2, end2 - 1);
    int len2 = lcs(begin1, end1 - 1, begin2, end2);
    return len1 > len2 ? len1 : len2;
  }
}
```

Ez a kód már jól működik, és már majdnem meg tudjuk határozni a leghosszabb közös részsorozatot:  
Azok az elemek, amelyek az algoritmus által megtalált leghosszabb közös részsorozatot alkotják, mindig szerepelnek az `if` elágazás első ágában - ha ezeket az elemeket összegyűjtjük, megkapunk egy lehetséges leghosszabb részsorozatot. Ezzel a megoldással az a probléma, hogy a rekurzív hívás miatt egy-egy elemet többször is vizsgál a függvény, így a végeredményben is többször szerepelne.  
A függvény másik problémája, hogy borzasztó lassú - pont mint abban a klasszikus esetben, amikor a Fibonacci számsor elemeit rekurzívan számoltuk ki. A megoldás itt is hasonló lesz: a részeredményeket egy táblázatban fogjuk eltárolni, és az alapján állapítjuk majd meg a végeredményt.

Folytatás hamarosan, nemsokára kitérünk arra is, hogy hogyan készítsük el ezt a táblázatos megoldást.