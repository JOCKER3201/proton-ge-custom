# Łatki zastąpione przez odtworzenie ścieżki EXT

## 0006-vkd3d-resolve-omm-handles-for-prebuild-sizing.patch

Dokładała rozwiązywanie uchwytów mikromap w ścieżce wymiarowania BLAS, żeby zapytanie o rozmiar
widziało ten sam łańcuch co budowa. Celowała w prawdziwy problem, ale obok: zamiast przestać
odraczać rozwiązywanie, dokładała drugi przebieg z inną polityką błędu dla wymiarowania (miękką)
i dla budowy (twardą).

Zastąpiona przez rozwiązywanie uchwytu wprost w konwerterze — tak jak robił to scalony backend
upstreamu (PR #2519). Cała warstwa odroczona dla EXT usunięta; ścieżka KHR zachowuje własną.

Nie usuwać tego pliku — to zapis rozumowania, nie martwy kod.

## 0005-vkd3d-ext-opacity-micromap-fallback.patch — wycofana 2026-09-06

Backend `VK_EXT_opacity_micromap`, odtworzony wiernie wobec scalonych PR #2507/2512/2515/2519
(patrz `PLAN-omm-ext-sciezka.md`, tutaj obok). Wycofana, bo **nie może zadziałać dla Cyberpunka
2077 na sterowniku bez `VK_KHR_opacity_micromap`**, a taki jest sterownik na tej maszynie
(NVIDIA 610.57.04 wystawia wyłącznie `VK_EXT_opacity_micromap` rev. 2).

Powód, wprost ze specyfikacji `GLSL_EXT_opacity_micromap_ray_query_mode`:

> If a ray query traversal encounters an acceleration structure that contains opacity micromaps
> when `gl_EnableOpacityMicromapEXT` is false, the behavior is undefined.

Zgodę wyraża tryb wykonania `OpacityMicromapIdKHR` (6031) + capability
`RayTracingOpacityMicromapExecutionModeKHR` (6032). Według `vk.xml` oba wymagają
`VK_KHR_opacity_micromap` i **nie mają odpowiednika w EXT**. Cyberpunk w path tracingu używa
ray query, więc na EXT każde przejście promienia to zachowanie niezdefiniowane → zawieszenie GPU.

Strona danych (budowa tablic, wpięcie do BLAS) była w EXT wyrażalna w całości i działała:
1571 tablic zbudowanych, zero błędów. Brakowało wyłącznie zgody po stronie shadera.

**Kod nie jest bezwartościowy:** dla OMM przez `TraceRay` w potokach RT opt-in wyraża flaga
tworzenia potoku, która w EXT istnieje. Łatka jest poprawna dla takich gier, bezużyteczna
dla ray query.

## 0003-vkd3d-witcher3-rtas-scratch-headroom.patch — wycofana na życzenie właściciela

