# Tunniste kehyksessä

## Kuvaus

Edellisessä tehtävässä totesit että yleislähetys menee kaikille, koska kytkimellä ei ole mitään perustetta rajata sitä. VLAN antaa sille perusteen: kehykseen lisätään neljä tavua, jotka kertovat mihin verkkoon kehys kuuluu. Tässä tehtävässä katsot mihin ne neljä tavua menevät.

Lue ennen aloitusta MikroTikin VLAN-sivulta kohta 802.1Q.

Tehtävässä kaapataan kehys reitittimellä ja luetaan sen tavut yksi kerrallaan. Kun tiedät, miltä tavallinen kehys näyttää, on helppo nähdä, mihin VLAN-tunniste sen sisällä sijoittuu ja mitä sille tapahtuu matkalla.

## Kaappaa yksi kehys ja katso sen tavut

Käytät samaa työkalua kuin edellisessä tehtävässä, mutta nyt pyydät näkyviin myös itse kehyksen sisällön.

1. Käynnistä MacBookilta muutama ping toisessa ikkunassa, ennen kuin käynnistät kaappauksen, jotta ehdit käynnistää molemmat lähes yhtä aikaa:

    ```
    ping <raspberryn-osoite>
    ```

    Ilman `-c`-valitsinta ping jatkuu, kunnes pysäytät sen. Näin kaapattavaa liikennettä on tarjolla koko ajan.

2. Anna sen jäädä pyörimään taustalle. Käynnistä sitten reitittimellä kaappaus, joka näyttää raa'at tavut:

    ```
    /tool sniffer quick interface=<bridgen-nimi> show-frame=yes
    ```

    `show-frame=yes` lisää jokaisen yhteenvetorivin alle kehyksen sisällön sellaisenaan.

3. Anna sen pyöriä muutama sekunti ja pysäytä se Ctrl+C:llä. Pysäytä myös ping MacBookilla.

4. Tuloste näyttää jokaisesta kaapatusta kehyksestä sekä yhteenvetorivin että sen alla itse tavut heksadesimaalimuodossa. Etsi joukosta yksi rivi, jonka PROTOCOL-sarakkeessa lukee icmp.

    Jos icmp-rivejä ei tule lainkaan, reitittimen kytkinpiiri välittää MacBookin ja Raspberry Pi:n väliset kehykset suoraan porttien välillä, eivätkä ne käy prosessorilla. Tämä on sama ilmiö kuin edellisen tehtävän lopussa. Pingaa silloin Raspberry Pi:n sijaan reitittimen omaa osoitetta. Reitittimelle osoitetut kehykset tulevat aina prosessorille asti, ja niiden rakenne on täsmälleen sama.

## Lue tavut

Kehyksen ensimmäiset tavut menevät aina samalla kaavalla:

- Tavut 1–6: vastaanottajan MAC-osoite
- Tavut 7–12: lähettäjän MAC-osoite
- Tavut 13–14: Type-kenttä

Heksadesimaalimuodossa jokainen tavu on kaksi merkkiä, esimerkiksi `a4` tai `08`. Kuusi tavua on siis kaksitoista merkkiä. Voit tarkistaa laskusi edellisen tehtävän muistiinpanoista: pingipyynnössä ensimmäiset kuusi tavua ovat Raspberry Pi:n MAC-osoite ja seuraavat kuusi MacBookin. Vastauksessa järjestys on toisin päin.

Etsi rivin hex-dumpista kohta `08 00`.

Se on Type-kenttä, ja se sijaitsee tasan 12 tavun jälkeen kehyksen alusta — laske itse tavut auki ja tarkista että osuma on oikeassa kohdassa, älä vain etsi ensimmäistä `08 00`-esiintymää tekstistä.

Sama tavupari voi esiintyä kehyksessä muuallakin. Pingipyynnön ICMP-otsikko alkaa tavuilla `08 00`, koska ICMP-tyyppi 8 tarkoittaa kaiutuspyyntöä ja koodi on 0. Myös MAC-osoitteessa voi sattumalta olla samat merkit. Vain paikka kertoo, mikä tavupari on Type-kenttä.

`0x0800` tarkoittaa IPv4:ää. Se kertoo vastaanottajalle mitä 14. tavun jälkeen tulee: IP-paketti.

Heti Type-kentän perässä on yleensä tavu `45`. Se on IP-otsikon ensimmäinen tavu: 4 on IP-versio ja 5 otsikon pituus. Siitä näet, että tulkitsit Type-kentän oikein.

Katso vielä yhteenvetorivistä kehyksen kokonaispituus tavuina, sarake on yleensä nimeltään SIZE tai LEN.

Pituus sisältää Ethernet-otsikon, IP-otsikon, ICMP-otsikon ja pingin datan. Kehyksen loppuun kuuluva neljän tavun tarkistussumma (FCS) ei yleensä näy kaappauksessa, koska verkkokortti poistaa sen ennen kuin kehys annetaan eteenpäin.

## Mihin tunniste menee

802.1Q ei lisää mitään kehyksen alkuun eikä loppuun. Se työntää neljä tavua keskelle, heti lähettäjän MAC-osoitteen jälkeen ja ennen vanhaa Type-kenttää.

| Tavut | Tavallinen kehys | Tunnisteellinen kehys |
| --- | --- | --- |
| 1–6 | Vastaanottajan MAC | Vastaanottajan MAC |
| 7–12 | Lähettäjän MAC | Lähettäjän MAC |
| 13–14 | Type `08 00` | `81 00` |
| 15–16 | IP-paketti alkaa | Prioriteetti ja VLAN-tunnus |
| 17–18 | IP-paketti jatkuu | Type `08 00` |
| 19– | IP-paketti jatkuu | IP-paketti alkaa |

Ne neljä tavua ovat:

- Kaksi ensimmäistä tavua saavat arvon `0x8100`. Ne ovat siinä kohdassa, jossa vastaanottaja odotti näkevänsä Type-kentän. Arvo `0x8100` ei tarkoita mitään protokollaa vaan kertoo: tässä kehyksessä on VLAN-tunniste, oikea Type-kenttä on neljä tavua myöhempänä.
- Kaksi seuraavaa tavua sisältävät prioriteetin ja VLAN-tunnuksen. Tunnukselle on varattu 12 bittiä.

Kaksi tavua on 16 bittiä. Niistä kolme ensimmäistä on prioriteetti, jolla esimerkiksi puheliikenne voidaan merkitä kiireellisemmäksi. Seuraava bitti merkitsee, saako kehyksen pudottaa ruuhkassa ensimmäisenä. Loput 12 bittiä ovat VLAN-tunnus.

Vanha Type-kenttä ei katoa, se vain siirtyy neljä tavua eteenpäin. Kehys kasvaa neljällä tavulla.

Ratkaisu on nokkela, koska vanhat laitteet eivät mene sekaisin. Laite, joka ei tunne VLANeja, näkee Type-kentän kohdalla tuntemattoman arvon ja jättää kehyksen käsittelemättä sen sijaan, että tulkitsisi tunnistetta väärin IP-paketiksi.

Laske ja kirjaa ylös:

1. Montako eri arvoa 12 bitillä voi esittää?
2. Kolme näistä arvoista on varattu eikä niitä käytetä tavallisissa verkoissa. MikroTikin sivulla sanotaan mitkä. Montako jää käyttöön?
3. Mikä oli ping-pakettisi pituus, ja mikä se olisi tunnisteen kanssa?

## Miksi tunniste ei mene työasemalle asti

1. Katso MacBookisi verkkoasetukset:

    ```
    ifconfig
    ```

    Ilman liitännän nimeä `ifconfig` listaa koneen kaikki liitännät, esimerkiksi `lo0` (kone itse), `en0` (yleensä Wi-Fi) ja kaapelisovittimen.

2. Etsi listalta liitäntä, jonka nimessä tai asetuksissa mainittaisiin VLAN.

    Sellaista ei ole.

    Koneesi ei ole missään kohtaa määritelty kuulumaan mihinkään VLANiin, eikä se osaa tulkita `0x8100`-arvoa miksikään.

    Jos sille lähetettäisiin tunnisteellinen kehys, se näkisi Type-kentän kohdalla arvon jota se ei tunne, eikä osaisi tehdä kehykselle mitään.

    macOS ja Linux osaisivat kyllä käsitellä tunnisteita, jos niille luotaisiin erillinen VLAN-liitäntä. Tavallisiin työasemiin sellaista ei kuitenkaan tehdä, ja koko järjestelmä on suunniteltu niin, ettei sitä tarvita.

3. Tästä seuraa sääntö, joka määrää kaiken myöhemmän työn: tunniste lisätään kun kehys siirtyy verkkolaitteiden välillä, ja se poistetaan ennen kuin kehys annetaan tavalliselle päätelaitteelle. Työasema ei tiedä olevansa missään VLANissa, eikä sen tarvitse tietää.

    VLAN on siis kytkimien välinen sopimus. Työasema kuuluu VLANiin sen perusteella, mihin kytkimen porttiin se on kytketty, ei omien asetustensa perusteella.

## Tarkista vielä yksi asia

1. Kehys kasvaa neljällä tavulla. Katso mitä liitäntäsi sallii:

    ```
    /interface ethernet print detail
    ```

    Tuloste on pitkä, koska jokaisesta Ethernet-portista tulostuu oma lohkonsa kaikkine asetuksineen. Riittää, että katsot yhtä porttia.

2. Etsi tulosteesta mtu ja l2mtu. Huomaa että l2mtu on suurempi kuin 1500. Ero on siinä, että mtu koskee IP-pakettia ja l2mtu koko kehystä — ja siihen on jätetty tilaa juuri tällaisia lisäyksiä varten.

    Tarkalleen ottaen MikroTik laskee l2mtu:hun kaiken paitsi kehyksen alun 14 tavua eli MAC-osoitteet ja Type-kentän. VLAN-tunniste lasketaan mukaan, joten 1500 tavun IP-paketti tunnisteen kanssa vaatii l2mtu:n vähintään 1504. Jos l2mtu olisi tasan 1500, täysimittainen paketti ei mahtuisi tunnisteen kanssa läpi.

## Ennen kuin jatkat

Sinun pitäisi pystyä kertomaan omin sanoin:

- Mihin kohtaan kehystä neljä tavua lisätään ja mikä kenttä siirtyy
- Mitä arvo `0x8100` kertoo vastaanottajalle
- Montako verkkoa yhdellä kaapelilla voi tunnisteen avulla erottaa
- Miksi työasemalle menevästä kehyksestä tunniste poistetaan

Seuraavassa tehtävässä katsot mitkä portit lisäävät tunnisteen ja mitkä poistavat sen.

### Lähteet

- [MikroTik: VLAN](https://manual.mikrotik.com/docs/bridging-and-switching/vlan)
- [MikroTik: Packet Sniffer](https://manual.mikrotik.com/docs/diagnostics-monitoring-and-troubleshooting/packet-sniffer)
- [MikroTik: MTU in RouterOS](https://manual.mikrotik.com/docs/hardware/mtu-in-routeros)
