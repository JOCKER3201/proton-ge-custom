# Plan: osobna ścieżka VK_EXT_opacity_micromap, wierna wobec upstreamu

Zrekonstruowane z PR #2507/#2512/#2515/#2519 (wszystkie scalone do vkd3d-proton
w czerwcu 2025, usunięte przy przejściu na KHR w #3069/#3071).
Licencja: LGPL-2.1 w drzewie LGPL-2.1. Przed commitem: audyt legalności.

---

# PLAN WDROŻENIA: osobna ścieżka VK_EXT_opacity_micromap, wierna wobec upstreamu, domyślnie wyłączona

---

## 1. KSZTAŁT

### 1.1 Zmienna środowiskowa: `VKD3D_CONFIG=dxr12`

Flaga **już istnieje** i jest martwa — `grep -rn "DXR_1_2"` po całym drzewie daje jedną linię:

```
/var/home/michael/.git/proton-11/vkd3d-proton/include/private/config_flag_decl.h:32
VKD3D_DECL_CONFIG("dxr12", DXR_1_2)
```

Zero użyć. To jest dokładnie ta nazwa, którą upstream trzymał jako `{"dxr12", VKD3D_CONFIG_FLAG_DXR_1_2}`, i nie ma powodu jej zmieniać ani dodawać nowej deklaracji.

**Zastrzeżenie do świadomej akceptacji:** u upstreamu `dxr12` znaczyło „włącz DXR 1.2", bo EXT był jedynym backendem. U nas DXR 1.2 działa domyślnie przez KHR, więc `dxr12` będzie faktycznie znaczyć „przełącz backend OMM na EXT". Nazwa jest wierna, semantyka przesunięta. Jeśli właściciel woli nazwę mówiącą prawdę, jedyna zmiana to jedna linia w `config_flag_decl.h`:

```c
VKD3D_DECL_CONFIG("omm_ext", DXR_1_2)
```

(symbol `DXR_1_2` zostaje, zmienia się tylko string rozpoznawany w `VKD3D_CONFIG`). Rekomenduję jednak zostawić `dxr12` — zlecenie mówi wprost o odtworzeniu tamtego kształtu.

### 1.2 Gdzie postawić przełącznik: JEDEN punkt, na rejestracji rozszerzenia

Master switch to `device.c:99`. Dziś:

```c
    VK_EXTENSION_DISABLE_COND(EXT_OPACITY_MICROMAP, EXT_opacity_micromap, VKD3D_CONFIG_FLAG_STATIC(NO_DXR)),
```

czyli opt-out, domyślnie **włączone** na każdym urządzeniu z EXT-em. Docelowo:

```c
    VK_EXTENSION_COND(EXT_OPACITY_MICROMAP, EXT_opacity_micromap, VKD3D_CONFIG_FLAG_STATIC(DXR_1_2)),
```

To jest jeden do jednego upstreamowe `VK_EXTENSION_COND(EXT_OPACITY_MICROMAP, EXT_opacity_micromap, VKD3D_CONFIG_FLAG_DXR_1_2)`, przełożone na naszą maszynerię flag (`VKD3D_CONFIG_FLAG_STATIC(CONF)` zamiast `1ull << 29`).

**Dlaczego to wystarcza jako kill switch.** Bez `dxr12` rozszerzenie nie trafia na listę włączanych → `vulkan_info->EXT_opacity_micromap == false` → struktura `opacity_micromap_features_ext` nie trafia do łańcucha `pNext` w `vkd3d_physical_device_info_init` → `opacity_micromap_features_ext.micromap == 0` → `d3d12_device_uses_ext_opacity_micromap()` (`vkd3d_private.h:6351`) zwraca `false` → **każda** gałąź `.ext` w drzewie jest martwa. Nie trzeba dokładać warunku w żadnym z pozostałych ~30 miejsc.

**Utrata bramki `NO_DXR`** jest pozorna: `VKD3D_CONFIG=nodxr` zeruje tier w `d3d12_device_determine_ray_tracing_tier`, więc `d3d12_device_supports_ray_tracing_tier_1_2()` (p. 2.1c) i tak zwróci `false`. Upstream miał tu identyczną „dziurę" i jej nie zamykał.

### 1.3 Rozdzielenie ścieżek KHR i EXT

Trzy poziomy, każdy z własnym predykatem — **nie mieszać ich**:

| pytanie | predykat | gdzie |
|---|---|---|
| „czy OMM jest w ogóle dostępny" (kod wspólny: bariery, bity usage, tier, publikacja do dxil-spirv) | `device->device_info.supports_opacity_micromap` | tak jak dziś, bez zmian |
| „czy działamy na backendzie EXT" (wybór gałęzi unii, dispatch, konwertery `_ext`) | `d3d12_device_uses_ext_opacity_micromap(device)` | tak jak dziś, bez zmian |
| „czy upstream w tym miejscu pytał o tier 1.2" (twarde błędy w konwersji, prebuild info, build path) | **nowy** `d3d12_device_supports_ray_tracing_tier_1_2(device)` | do dopisania |

**Decyzja architektoniczna nr 1 — zostawiamy DWA typy widoku.** `VKD3D_VIEW_TYPE_ACCELERATION_STRUCTURE` + `VKD3D_VIEW_TYPE_OPACITY_MICROMAP_EXT` (`vkd3d_private.h:1376-1380`) nie wracają do upstreamowego wspólnego `..._ACCELERATION_STRUCTURE_OR_OPACITY_MICROMAP` + `bool rtas_is_micromap`. Powód: czwarty argument `vkd3d_view_map_create_view2` jest u nas zajęty przez `enum vkd3d_rtas_kind`, który obsługuje niepowiązaną sprawę (BLP przy SERIALIZATION) i jest potrzebny ścieżce KHR. Scalenie typów zmieniłoby hashowanie **wszystkich** widoków AS, także pod KHR.

**Cena tej decyzji jest jedna i trzeba ją zapłacić jawnie:** upstreamowy niezmiennik „jeden VA = jeden rodzaj obiektu, na zawsze" u nas nie wynika z konstrukcji. Musi go wymusić kod — dokładnie tam, gdzie #2515 wstawił rozgałęzienie (p. 2.5). Bez tego kroku cała reszta planu nie ma sensu.

**Decyzja architektoniczna nr 2 — unie `vkd3d_omm_*` zostają.** `union vkd3d_omm_build_info`, `union vkd3d_omm_usage_info`, `union vkd3d_omm_triangles_info` (`vkd3d_private.h:368-378`, `:3251`, `:3260-3266`) to nasz nośnik dwóch backendów. Wymóg: **wnętrze gałęzi `.ext` musi być dosłownie upstreamowe**. Unia jest opakowaniem, nie zmianą semantyki.

### 1.4 Przełącznik musi też odebrać pierwszeństwo KHR-owi

Sam gate na rejestracji nie wystarczy do A/B: gdy urządzenie ma oba rozszerzenia, sonda w `device.c:2172` wyłączy EXT, więc `VKD3D_CONFIG=dxr12` nie dałby ścieżki EXT. Poprawka — przed sondą:

```c
    /* VKD3D_CONFIG=dxr12 wybiera backend VK_EXT_opacity_micromap. Zdejmujemy KHR w całości, żeby
     * urządzenie wyglądało możliwie blisko urządzenia EXT-only: struktura cech KHR nigdy nie trafia
     * do łańcucha (patrz warunek w linii 2699), więc using_khr_opacity_micromap wychodzi false bez
     * dotykania bloku rozstrzygającego. */
    if (VKD3D_CONFIG_FLAG_IS_SET(DXR_1_2))
        vulkan_info->KHR_opacity_micromap = false;
    else if (vulkan_info->KHR_opacity_micromap && vulkan_info->EXT_opacity_micromap)
    {
        /* ... istniejąca sonda bez zmian ... */
    }
```

Zweryfikowane: `KHR_opacity_micromap` jest czytane tylko w czterech miejscach (`device.c:98, 2172, 2191, 2699`), a `:2699` to właśnie warunek doczepiania struktury cech. Blok rozstrzygający (`device.c:2802-2806`) nie wymaga żadnej zmiany.

---

## 2. PLIK PO PLIKU

### 2.1 `libs/vkd3d/device.c`

**(a) Rejestracja rozszerzenia** — `device.c:99`, zamiana opisana w 1.2. Linia 98 (KHR) **bez zmian**.

**(b) Odebranie pierwszeństwa KHR** — `device.c:2170-2172`, wstawka z 1.4.

**(c) Nowy helper tieru** — obok `d3d12_device_supports_ray_tracing_tier_1_0` (`device.c:1806`). Upstream:

```c
bool d3d12_device_supports_ray_tracing_tier_1_2(const struct d3d12_device *device)
{
    return device->device_info.opacity_micromap_features.micromap &&
            device->d3d12_caps.options5.RaytracingTier >= D3D12_RAYTRACING_TIER_1_2;
}
```

**Odstępstwo wymuszone:** u nas jedna struktura cech nie wystarcza (mamy dwie). Wersja do wstawienia:

```c
bool d3d12_device_supports_ray_tracing_tier_1_2(const struct d3d12_device *device)
{
    return device->device_info.supports_opacity_micromap &&
            device->d3d12_caps.options5.RaytracingTier >= D3D12_RAYTRACING_TIER_1_2;
}
```

Deklaracja obok `vkd3d_private.h:6347`.

**Uwaga niezmiennika (upstream nazywał to wprost):** tej funkcji **nie wolno** użyć w `vkd3d_memory_info_init` — `d3d12_caps` nie są tam jeszcze wypełnione. `resource.c:11281` ma już poprawny komentarz o tym; nie regresować.

**(d) Promocja tieru** — `device.c:10250` bez zmian:

```c
    if (tier == D3D12_RAYTRACING_TIER_1_1 && info->supports_opacity_micromap)
    {
        INFO("DXR 1.2 support enabled.\n");
        tier = D3D12_RAYTRACING_TIER_1_2;
    }
```

Upstream promował bezwarunkowo na bicie cechy i to samo robimy; różnica jest tylko taka, że pod EXT ten bit zapala się teraz wyłącznie z `dxr12`.

**(e) Prebuild info OMM** — `d3d12_device_get_raytracing_opacity_micromap_array_prebuild_info_ext` (`device.c:8954-8996`) jest już praktycznie znak w znak upstreamowe. **Nic nie zmieniać.** Jedyna ingerencja w tym pliku to punkt (f).

**(f) Usunąć odroczone rozwiązywanie uchwytów ze ścieżki wymiarowania BLAS** — `device.c:9132-9136`. To warstwa z patcha `0006`, nie ma odpowiednika u upstreamu (patrz sekcja 3).

**(g) Zdjąć diagnostykę** — `device.c:1457-1471` (wymuszenie `FAULT` i `OMM_EXT_LINKAGE_USAGE_COUNTS`).

---

### 2.2 `libs/vkd3d/opacity_micromap.c` — ciężar prac

#### (a) Konwersja wpięcia BLAS — `vkd3d_opacity_micromap_convert_opacity_micromap_ext` (:630)

Cztery zmiany, wszystkie w kierunku „z powrotem do upstreamu".

**(a.1) Usunąć odpięcie przy `OpacityMicromapArray == 0`.** Do wycięcia (`:641-646`):

```c
        if (!geom_desc->OmmTriangles.pOmmLinkage ||
                !geom_desc->OmmTriangles.pOmmLinkage->OpacityMicromapArray)
        {
            memset(omm_triangles_info, 0, sizeof(*omm_triangles_info));
            return true;
        }
```

Upstream doczepia strukturę **bezwarunkowo** i wypełnia ją bezwarunkowo, a warunkiem obejmuje wyłącznie samo placement:

```c
                    geometry_infos[i].geometry.triangles.pNext = omm = &omm_infos[i];
                    omm->sType = VK_STRUCTURE_TYPE_ACCELERATION_STRUCTURE_TRIANGLES_OPACITY_MICROMAP_EXT;
                    omm->pNext = NULL;
```

```c
                    omm->indexBuffer.deviceAddress = geom_desc->OmmTriangles.pOmmLinkage->OpacityMicromapIndexBuffer.StartAddress;
                    omm->indexStride = geom_desc->OmmTriangles.pOmmLinkage->OpacityMicromapIndexBuffer.StrideInBytes;
                    omm->baseTriangle = geom_desc->OmmTriangles.pOmmLinkage->OpacityMicromapBaseLocation;

                    if (geom_desc->OmmTriangles.pOmmLinkage->OpacityMicromapArray)
                    {
                        omm->micromap = vkd3d_va_map_place_opacity_micromap(
                                &device->memory_allocator.va_map, device,
                                geom_desc->OmmTriangles.pOmmLinkage->OpacityMicromapArray);

                        if (omm->micromap == VK_NULL_HANDLE)
                            ERR("Failed to place OMM at VA 0x%"PRIx64".\n", geom_desc->OmmTriangles.pOmmLinkage->OpacityMicromapArray);
                    }
```

**Odstępstwo wymuszone:** nazwa rozwiązywacza w naszym drzewie to `vkd3d_va_map_place_opacity_micromap_ext`. Poza nazwą — kopiuj dosłownie, łącznie z brakiem sprawdzenia `pOmmLinkage` na NULL (upstream ufa runtime'owi D3D12).

**(a.2) Nieudane placement = tylko log, nie porzucenie budowy.** Do wycięcia (`:706-711`) `return false`; zostaje `ERR(...)` i przelot dalej, jak wyżej.

**(a.3) Przyjąć `DXGI_FORMAT_R8_UINT`.** W `vkd3d_..._index_type_ext` (`:615-628`) zamiast odrzucenia:

```c
                        case DXGI_FORMAT_R8_UINT:
                            FIXME_ONCE("Using UINT8 as OMM index format is technically out of spec.\n");
                            omm->indexType = VK_INDEX_TYPE_UINT8;
                            break;
```

**(a.4) Usunąć publikację histogramu na wpięciu** (`:684-686`) — patrz sekcja 3.

**Mechanizm doczepiania — odstępstwo wymuszone, ZOSTAJE.** Upstream robi bezpośrednie `geometry_infos[i].geometry.triangles.pNext = omm = &omm_infos[i];`. My możemy to zrobić w konwerterze tak samo (`vk_prepend_struct` jest zbędne, bo `pNext` jest w tym momencie zerem po memsecie), **ale pętla naprawcza w `command.c:21475-21494` musi zostać**. U nas tablica linkage rośnie inkrementalnie przez `vkd3d_array_reserve` między kolejnymi buildami w tej samej partii, więc wskaźnik zapisany przy pierwszym buildzie zwisa po realokacji przy drugim. W #2519 tej pętli nie widać — i nie zmyślam, że tam była (patrz sekcja 5).

#### (b) `vkd3d_opacity_micromap_write_postbuild_info_ext` (:451) → kształt upstreamu

Zmiana sygnatury: **nie-`static`**, z parametrem `desc_offset`:

```c
void vkd3d_opacity_micromap_write_postbuild_info(
        struct d3d12_command_list *list,
        const D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_DESC *desc,
        VkDeviceSize desc_offset,
        VkMicromapEXT vk_opacity_micromap)
```

(u nas z sufiksem `_ext`). Arytmetyka offsetu:

```c
    vk_buffer = resource->vk_buffer;
    offset = desc->DestBuffer - resource->va;
    offset += desc_offset;
```

Rozgałęzienie `switch` zamiast naszego `if/else if/else`, z **realnym** zapytaniem o SERIALIZATION:

```c
    switch (desc->InfoType)
    {
        case D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_COMPACTED_SIZE:
            vk_query_type = VK_QUERY_TYPE_MICROMAP_COMPACTED_SIZE_EXT;
            type_index = VKD3D_QUERY_TYPE_INDEX_OMM_COMPACTED_SIZE;
            stride = sizeof(uint64_t);
            break;
        case D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_SERIALIZATION:
            vk_query_type = VK_QUERY_TYPE_MICROMAP_SERIALIZATION_SIZE_EXT;
            type_index = VKD3D_QUERY_TYPE_INDEX_OMM_SERIALIZE_SIZE;
            stride = sizeof(uint64_t);
            break;
        default:
            FIXME("Unsupported InfoType %u.\n", desc->InfoType);
            /* TODO: CURRENT_SIZE is something we cannot query in Vulkan, so
                * we'll need to keep around a buffer to handle this.
                * For now, just clear to 0. */
            VK_CALL(vkCmdFillBuffer(list->cmd.vk_command_buffer, vk_buffer, offset,
                    sizeof(uint64_t), 0));
            return;
    }
```

(wcięcie komentarza TODO jest w upstreamie przesunięte o jedną spację — odtwarzam wiernie)

i ogon:

```c
    if (desc->InfoType == D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_SERIALIZATION)
    {
        VK_CALL(vkCmdFillBuffer(list->cmd.vk_command_buffer, vk_buffer, offset + sizeof(uint64_t),
                sizeof(uint64_t), 0));
    }
```

**Odstępstwo wymuszone — `d3d12_command_list_reset_query` NIE dodawać.** Upstream (czerwiec 2025) wołał go między alokacją slotu a `vkCmdWriteMicromapsPropertiesEXT`. Nasza baza jest nowsza i nasz `vkd3d_acceleration_structure_write_postbuild_info` (`acceleration_structure.c:442-450`) też go nie ma. Zasada: **ścieżka OMM robi dokładnie to, co robi obok ścieżka AS w naszym drzewie**, cokolwiek to jest. Dodanie resetu „bo tak było u upstreamu" byłoby wprowadzeniem obcego elementu do nowszej maszynerii pul wirtualnych.

#### (c) `vkd3d_opacity_micromap_copy_ext` (:825) → kształt upstreamu

Sygnatura wraca do uchwytu (widok był potrzebny tylko dla histogramu, który znika):

```c
void vkd3d_opacity_micromap_copy(
        struct d3d12_command_list *list,
        D3D12_GPU_VIRTUAL_ADDRESS dst, VkMicromapEXT src_omm,
        D3D12_RAYTRACING_ACCELERATION_STRUCTURE_COPY_MODE mode)
{
    const struct vkd3d_vk_device_procs *vk_procs = &list->device->vk_procs;
    VkCopyMicromapInfoEXT info;
    VkMicromapEXT dst_omm;

    dst_omm = vkd3d_va_map_place_opacity_micromap(&list->device->memory_allocator.va_map, list->device, dst);
    if (dst_omm == VK_NULL_HANDLE)
    {
        ERR("Invalid dst address #%"PRIx64" for OMM copy.\n", dst);
        return;
    }

    info.sType = VK_STRUCTURE_TYPE_COPY_MICROMAP_INFO_EXT;
    info.pNext = NULL;
    info.dst = dst_omm;
    info.src = src_omm;
    if (convert_copy_mode(mode, &info.mode))
        VK_CALL(vkCmdCopyMicromapEXT(list->cmd.vk_command_buffer, &info));
}
```

Uwagi do odtworzenia: **bez memsetu**, pola pojedynczo, `info.mode` wypełnia konwerter dopiero w warunku, a **kolejność jest odwrotna niż nasza** — najpierw placement `dst`, dopiero potem konwersja trybu. Nasz obecny kod sprawdza tryb przed placementem „żeby nie zostawiać mikromapy na VA, które nic nie dostanie"; to jest ostrożniejsze, ale to jest odstępstwo i przy odtwarzaniu 1:1 znika.

Konwerter (nasz `convert_copy_mode_ext`) do wyrównania: `FIXME` zamiast `FIXME_ONCE`, tekst `"Unsupported OMM copy mode #%x.\n"`.

#### (d) `vkd3d_opacity_micromap_emit_immediate_postbuild_info_ext` — trymowanie sygnatury

Upstream:

```c
void vkd3d_opacity_micromap_emit_immediate_postbuild_info(
        struct d3d12_command_list *list, uint32_t count,
        const D3D12_RAYTRACING_ACCELERATION_STRUCTURE_POSTBUILD_INFO_DESC *desc,
        VkMicromapEXT vk_opacity_micromap)
```

Nasza wersja ma dodatkowy `VkDeviceAddress va` (do `RT_TRACE`) — usunąć wraz z tym `RT_TRACE`. Ciało (bariera + pętla + `end_barrier`) jest już bit w bit upstreamowe; wywołanie w pętli ma iść z `desc_offset` = 0:

```c
    for (i = 0; i < count; i++)
        vkd3d_opacity_micromap_write_postbuild_info(list, &desc[i], 0, vk_opacity_micromap);
```

**Pułapka nazewnicza do rozstrzygnięcia w kodzie:** nasza bezsufiksowa `vkd3d_opacity_micromap_emit_immediate_postbuild_info` (:414) jest KHR-owa (bierze `VkAccelerationStructureKHR`, deleguje do `vkd3d_acceleration_structure_write_postbuild_info`). Upstreamowa funkcja o **identycznej nazwie** bierze `VkMicromapEXT`. Nie scalać, nie mylić.

#### (e) Do usunięcia z tego pliku (szczegóły w sekcji 3)

`:226-343` (cała warstwa histogramu), `:722-798` (`resolve_omm_va_maps` i `_ext`), `:660-675` (DIAG-ADDR), `:688-705` (komentarze POMIAR 11/12), `:782` (POMIAR5-WPIECIE), `:359` (bramka `OMM_EXT_LINKAGE_USAGE_COUNTS`).

#### (f) Zostawić bez zmian

`vkd3d_opacity_micromap_convert_inputs_ext` (:195) — semantycznie identyczne z upstreamem, w tym `usageCountsCount`/`pUsageCounts` i wspólność ze ścieżką wymiarowania (niezmiennik I2). Wyniesienie pętli do `..._convert_inputs_usages_ext` (:169) to kształt, nie zachowanie. `vkd3d_opacity_micromap_end_barrier` (:393) — bajt w bajt upstream. Bariera w emit (:534-541) — te same bity.

Kosmetyka do rozważenia dla czystości diffu wobec upstreamu: `d3d12_build_flags_to_vk_ext` (:48) ma nasze `FIXME_ONCE` o ignorowanym `ALLOW_UPDATE`, którego upstream nie miał. Log-only, można zostawić — ale odnotować.

---

### 2.3 `libs/vkd3d/acceleration_structure.c`

#### (a) BRAKUJĄCY HUNK #2515 — najważniejsza wstawka całego planu

`vkd3d_acceleration_structure_emit_postbuild_info` (`:515`) dziś nie zna mikromap w ogóle:

```c
    vkd3d_va_map_try_read_rtas(&list->device->memory_allocator.va_map, list->device, rtas_va,
            &vk_acceleration_structure, &rtas_kind);

    if (vk_acceleration_structure == VK_NULL_HANDLE)
    {
        WARN("Emit postbuild placing unknown AS at #%" PRIx64 " future BLP queries may not be reliable.\n", rtas_va);
        rtas_kind = VKD3D_RTAS_KIND_UNKNOWN;
        vk_acceleration_structure =
                vkd3d_va_map_place_acceleration_structure(&list->device->memory_allocator.va_map, list->device,
                rtas_va, rtas_kind);
    }
```

Aplikacja wołająca `EmitRaytracingAccelerationStructurePostbuildInfo` na VA tablicy OMM trafia tu, `try_read_rtas` chybia (bo pod tym VA leży widok typu `OPACITY_MICROMAP_EXT`), a `place_acceleration_structure` **zakłada `VkAccelerationStructureKHR` na tym samym zakresie bufora, na którym już siedzi `VkMicromapEXT`**, po czym leci `vkCmdWriteAccelerationStructuresPropertiesKHR` na strukturze, która nigdy nie była budowana. Jedyny ślad to `WARN`.

Wstawka **przed** `try_read_rtas`, wzorowana na #2515 (najpierw odczyt, dopiero potem placement):

```c
    if (d3d12_device_uses_ext_opacity_micromap(list->device))
    {
        VkMicromapEXT vk_opacity_micromap;

        vk_opacity_micromap = vkd3d_va_map_try_read_opacity_micromap_ext(
                &list->device->memory_allocator.va_map, list->device, rtas_va);

        if (vk_opacity_micromap != VK_NULL_HANDLE)
        {
            vkd3d_opacity_micromap_write_postbuild_info_ext(list, desc, 0, vk_opacity_micromap);
            return;
        }
    }
```

**Odstępstwa wymuszone i ich uzasadnienie:**
- upstream miał tu jedno `vkd3d_va_map_try_read_rtas(..., &vk_acceleration_structure, &vk_opacity_micromap)` z dwoma wyjściami; u nas nie ma jednego wywołania zwracającego oba, bo dyskryminator nie siedzi w widoku tylko w typie klucza — stąd dwie próby zamiast jednej;
- upstream przekazywał `i * stride` jako `desc_offset`, bo jego funkcja obsługiwała tablicę VA; nasza obsługuje jeden VA i offset liczy się z `desc->DestBuffer`, stąd `0`;
- upstream bramkował na `d3d12_device_supports_ray_tracing_tier_1_2`; u nas gałąź jest EXT-specyficzna, więc predykatem jest backend.

**Nie dodawać upstreamowego `vkd3d_opacity_micromap_emit_postbuild_info` (wariant po tablicy VA).** W #2512 był martwy (zero wywołań), a #2515 wpiął OMM przez `write_postbuild_info` bezpośrednio, dokładnie tak jak wyżej. Przenoszenie martwego kodu to nie wierność.

#### (b) `ALLOW_OMM_LINKAGE_UPDATE` — dołożyć drugi bit

Dziś (`:39-43`) mapujemy na jeden bit, upstream na dwa:

```c
    if (flags & D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_ALLOW_OMM_LINKAGE_UPDATE)
    {
        /* D3D12 spec isn't clear on what is allowed to be updated and when */
        vk_flags |= VK_BUILD_ACCELERATION_STRUCTURE_ALLOW_OPACITY_MICROMAP_UPDATE_BIT_EXT |
                VK_BUILD_ACCELERATION_STRUCTURE_ALLOW_OPACITY_MICROMAP_DATA_UPDATE_BIT_EXT;
    }
```

**Problem strukturalny do rozwiązania w kodzie:** nasze `d3d12_build_flags_to_vk` (`acceleration_structure.c:24`) **nie ma parametru `device`** — sprawdzone. Bit `..._DATA_UPDATE_BIT_*` wolno ustawić tylko przy włączonym rozszerzeniu. Dwa wyjścia: (1) dodać parametr `const struct d3d12_device *device` i bramkować na `supports_opacity_micromap`, (2) zostawić bez bramki, licząc na to, że sam bit bez rozszerzenia jest ignorowany — **niedopuszczalne**, to jest naruszenie VU. Wybrać (1). Bity KHR i EXT są aliasami, więc pisownia `_KHR` (jak w naszym pliku) jest równoważna.

#### (c) `flush_postbuild_batch` (:545) — zostaje, ale z poprawką

Cała warstwa batchowania postbuildu jest naszą nadbudową (upstream emitował natychmiast). **Dla ścieżki `Emit...PostbuildInfo` zostaje** — jest częścią naszej nowszej bazy dla AS i nie ma powodu jej rozbierać. Ale:

- klasyfikacja OMM ma się dziać **w punkcie użycia**, w `vkd3d_acceleration_structure_emit_postbuild_info` (p. 2.3a), tak jak w #2515 — nie przy kolejkowaniu w `command.c`;
- pętla `has_incoming_uav` (`:549-550`) czyta `infos[i].rtas_vk.rtas` także dla wpisów OMM, czyli czyta nieaktywną gałąź unii. Na 64 bitach działa przypadkiem; przy okazji tej pracy warto to rozstrzygnąć jawnie przez `is_omm`.

#### (d) Bez zmian

`vkd3d_acceleration_structure_convert_triangles` (:71) — znak w znak #2519, poza `TRACE` zamiast `WARN` (:85). `get_geometry_count` (:62) — identyczne. Kolejność w `case OMM_TRIANGLES` (:285-297) — identyczna.

---

### 2.4 `libs/vkd3d/command.c`

**(a) Postbuild po budowie OMM — z powrotem natychmiastowy.** Do wycięcia `:21739-21755` (kolejkowanie z `info->is_omm = true`), w zamian upstream:

```c
    if (num_postbuild_info_descs)
    {
        /* This doesn't seem to get used very often, so just record the build command
         * for now. If this ever becomes a performance issue, we can add postbuild info
         * to the batch. */
        d3d12_command_list_flush_rtas_batch(list);

        vkd3d_opacity_micromap_emit_immediate_postbuild_info(list,
                num_postbuild_info_descs, postbuild_info_descs,
                build_info->dstMicromap);
    }
```

(u nas z sufiksem `_ext` i po trymowaniu sygnatury z 2.2d). Sens: build mikromapy jest już nagrany w buforze poleceń, zanim leci `vkCmdWriteMicromapsPropertiesEXT`.

**(b) Uchwyt celu — z powrotem wprost z `va_map`.** Dziś (`:21674-21684`) bierzemy go przez widok, bo widok niósł histogram. Po usunięciu histogramu:

```c
    if (desc->DestAccelerationStructureData)
    {
        build_info->dstMicromap =
                vkd3d_va_map_place_opacity_micromap(&list->device->memory_allocator.va_map,
                        list->device, desc->DestAccelerationStructureData);
        if (build_info->dstMicromap == VK_NULL_HANDLE)
        {
            ERR("Failed to place destMicromap. Dropping call.\n");
            return;
        }
    }
```

**Odstępstwo świadome, do zachowania:** nasz `d3d12_command_list_discard_omm_build_info_ext` (:21408) ma zostać wołany przed każdym takim `return`. Upstream przy błędzie po alokacji po prostu wracał, zostawiając w partii wpis częściowo wypełniony — to jest jego błąd, nie cecha, i jego odtwarzanie nie ma żadnej wartości poznawczej.

**Odstępstwo świadome nr 2, do przedyskutowania:** dziś przy `DestAccelerationStructureData == 0` mamy twardy `ERR("OMM array build without a destination address. Dropping call.\n")` (`:21664-21671`). Upstream w tym przypadku **nagrywa build z `dstMicromap == VK_NULL_HANDLE`**. To jest gwarantowane naruszenie VU i realny kandydat na device-lost. Rekomendacja: odtworzyć upstreamowe `if (...)`, ale zachować nasz drop przy zerowym VA jako jedyną, jawnie opisaną w komentarzu poprawkę bezpieczeństwa. Jeśli właściciel chce czystej wierności — to jest jedna linia do wywalenia i pierwszy punkt do sprawdzenia, gdyby hang wrócił.

**(c) Kolejność dispatchy we `flush_rtas_batch` — odwrócić z powrotem.** Dziś `vkCmdBuildMicromapsEXT` (:21577) idzie **przed** `vkCmdBuildAccelerationStructuresKHR` (:21583). Upstream:

```c
    if (rtas_batch->build_info_count)
        VK_CALL(vkCmdBuildAccelerationStructuresKHR(list->cmd.vk_command_buffer,
                rtas_batch->build_info_count, rtas_batch->build_infos, rtas_batch->range_ptrs));

    if (rtas_batch->omm_build_info_count)
        VK_CALL(vkCmdBuildMicromapsEXT(list->cmd.vk_command_buffer,
                rtas_batch->omm_build_info_count, rtas_batch->omm_build_infos));
```

**Odstępstwo wymuszone:** nasz warunek OMM musi zachować bramkę backendu (`&& d3d12_device_uses_ext_opacity_micromap(list->device)`) i indeksowanie przez `.ext`.

**(d) Bariera przy zmianie `build_type` — dołożyć brakujące połówki.** Upstream:

```c
        if (list->device->device_info.opacity_micromap_features.micromap)
        {
            vk_barrier.srcAccessMask |= VK_ACCESS_2_MICROMAP_READ_BIT_EXT | VK_ACCESS_2_MICROMAP_WRITE_BIT_EXT;
            vk_barrier.dstAccessMask |= VK_ACCESS_2_MICROMAP_READ_BIT_EXT | VK_ACCESS_2_MICROMAP_WRITE_BIT_EXT;
        }
```

Nasz `d3d12_command_list_flush_rtas_barrier` (:21625-21630) dokłada `MICROMAP_WRITE` tylko do src i `MICROMAP_READ` tylko do dst. Wyrównać do obu par, bramka `supports_opacity_micromap`.

**(e) Usunąć publikację histogramu** — `:21689`.

**(f) Usunąć wywołanie odroczonego resolve** — `:21906-21907`.

**(g) Zostawić bez zmian:**
- `d3d12_command_list_allocate_omm_build_info_ext` (:21375) — zgodne z #2512;
- pętla przeliczająca `pUsageCounts` w `fixup` (:21464-21472) — to jest **upstreamowy niezmiennik I1**, tylko przeniesiony z `flush` do `fixup`; bez niej wskaźnik zwisa po realokacji `omm_usage_infos`;
- pętla przepinająca `pNext` (:21475-21494) — odstępstwo wymuszone, uzasadnione w 2.2a;
- blok breadcrumbów (:21693-21737) — zgodny co do znaku;
- trzy wstawki barierowe w `vk_access_and_stage_flags_from_d3d12_resource_state` (:6465-6471, :6484-6490, :6518-6520) — te same bity, ta sama kolejność co #2507;
- kształt dyspozytora w `CopyRaytracingAccelerationStructure` (:22214-22258) — dwie próby zamiast upstreamowej jednej są strukturalnie konieczne przy dwóch typach widoku; kolejność (OMM najpierw) zachować.

**(h) Zdjąć diagnostykę** — `:21673` (`POMIAR5-BUDOWA`).

---

### 2.5 `libs/vkd3d/va_map.c`

**Nic nie dodawać.** Upstreamowe `vkd3d_va_map_try_read_rtas` z parametrem `VkMicromapEXT *micromap` nie ma u nas odpowiednika w tej sygnaturze, ale jego **funkcja** jest już pokryta przez `vkd3d_va_map_try_read_opacity_micromap_ext` (:314). Brakowało konsumenta, nie producenta — konsumenta dokłada 2.3a.

Do zdjęcia: komentarze POMIAR 3 i POMIAR 4 (`:472-480`).

Odnotować jako potwierdzoną zgodność: `key.u.buffer.size = resource->size - key.u.buffer.offset;` (:471) to **dokładnie** idiom upstreamu. Hipoteza z POMIARU 3 jest tu ostatecznie zamknięta.

---

### 2.6 `libs/vkd3d/resource.c`

**Nic nie zmieniać poza zdjęciem diagnostyki** (`:5389-5398`, `POMIAR6-*`).

Uzasadnienie: `vkd3d_view_map_get_view` (:1563-1578) jest znak w znak z #2515. Bity usage w `vkd3d_create_buffer` (:235-243) mają układ identyczny z #2507. Sonda w `vkd3d_memory_info_init` (:11286-11293) utrzymuje niezmiennik „sonda ⊇ realne bufory". `VkMicromapCreateInfoEXT` z memsetem jest funkcjonalnie równoważne upstreamowemu wypełnianiu pole po polu (`createFlags = 0`, `deviceAddress = 0` wychodzą z memsetu).

Jedyna zmiana wynikowa: po usunięciu histogramu z `vkd3d_create_opacity_micromap_view_ext` (:5369) znika przypisanie `object->info.buffer.omm_histogram_ext = NULL;`, a z `vkd3d_view_destroy` (:4975-4981) znika pętla zwalniająca listę.

---

### 2.7 `libs/vkd3d/vkd3d_private.h`

- deklaracja `bool d3d12_device_supports_ray_tracing_tier_1_2(const struct d3d12_device *device);` obok `:6347`;
- usunąć `struct vkd3d_omm_usage_histogram_ext` (:386-391) i pole `omm_histogram_ext` z `info.buffer` (:1405-1409);
- deklaracje `_ext` postbuildu/copy dostosować do nowych sygnatur (p. 2.2b, 2.2c, 2.2d);
- `struct vk_acceleration_structure_postbuild_info` (:3221-3235) **zostaje** — jest potrzebna ścieżce `Emit...` dla AS; znika tylko użycie `is_omm` przy budowie OMM.

---

### 2.8 Pliki, w których **nie ma nic do zrobienia**

- `libs/vkd3d/vulkan_procs.h` (:227-232) — dokładnie te same 6 PFN-ów, w tej samej kolejności co #2507.
- `libs/vkd3d/breadcrumbs.c` (:76, :78) — `"build_omm"`, `"copy_omm"` na właściwych pozycjach.
- `libs/vkd3d/meson.build:91` — `'opacity_micromap.c'` obecne.
- `libs/vkd3d/raytracing_pipeline.c` (:2551-2552) — `VK_PIPELINE_CREATE_2_RAY_TRACING_OPACITY_MICROMAP_BIT_KHR` w `flags2` to alias upstreamowego `VK_PIPELINE_CREATE_RAY_TRACING_OPACITY_MICROMAP_BIT_EXT` w `flags`.
- `libs/vkd3d-shader/dxil.c` (:1160-1174) — **nie cofać do `#if 0`**. Upstream miał tam stub tylko dlatego, że dxil-spirv nie znał wtedy tej opcji. Nasz blok jest aktywny i bogatszy o dwa pola (`trace_ray_enabled`, `ray_query_force_omm_execution_mode_in_legacy_sm`), a ścieżka jest wspólna dla obu backendów.
- `libs/vkd3d/device_vkd3d_ext.c`, `command_list_vkd3d_ext.c` — warstwa NVAPI, cała seria #2505–#2519 jej nie zna. Nie ruszać.

---

## 3. CZEGO NIE PRZENOSIĆ — adaptacje pod KHR do usunięcia

Kolejność = malejąca podejrzliwość, czyli sugerowana kolejność bisekcji.

**1. Odpięcie wpięcia przy `OpacityMicromapArray == 0`** — `opacity_micromap.c:641-646`. Zmienia **geometrię widzianą przez sterownik**: BLAS bez wpięcia to inny obiekt niż BLAS z wpięciem o pustym uchwycie. Upstream doczepiał zawsze.

**2. Brak gałęzi OMM w ścieżce `Emit...PostbuildInfo`** — brak w `acceleration_structure.c:515`. Pozwala położyć `VkAccelerationStructureKHR` na buforze mikromapy. Nie jest to „adaptacja pod KHR" w sensie przeniesionego kodu, tylko luka powstała z rozdzielenia typów widoku — ale skutek jest ten sam i naprawa należy do tej listy.

**3. Cała warstwa odroczonego rozwiązywania uchwytów.** Do usunięcia:
- `vkd3d_acceleration_structure_resolve_omm_va_maps` (`opacity_micromap.c:722`) i `..._ext` (`:756-798`) wraz z parametrem `bool best_effort`;
- wywołania: `command.c:21906-21907` (twardo) i `device.c:9132-9136` (miękko);
- patch `/var/home/michael/.git/proton-11/patches/vkd3d-proton/0006-vkd3d-resolve-omm-handles-for-prebuild-sizing.patch` (6563 B) — w odtworzonej ścieżce **nie ma dla niego miejsca**.

To jest ten kształt, o którym mówi hipoteza właściciela: bezpieczny pod KHR (mikromapa adresowana wskaźnikiem), bezsensowny pod EXT. Konwerter już rozwiązuje uchwyt na miejscu, więc odroczenie jako takie zostało zdjęte — ale **drugi przebieg został obok i nadpisuje to samo pole z inną polityką błędu dla wymiarowania i dla budowy**, co jest wprost zabronione przez upstreamowy niezmiennik I2/N8.

**4. `return false` przy nieudanym placement** — `opacity_micromap.c:706-711`. Upstream: `ERR` i jazda dalej.

**5. Publikacja `usageCounts` na wpięciu BLAS** — cała warstwa `opacity_micromap.c:226-343` plus wywołania `:684-686`, `:21689`, plus pole `omm_histogram_ext`, plus pętla zwalniająca w `resource.c:4975-4981`, plus flaga `VKD3D_DECL_CONFIG("omm_ext_linkage_usage_counts", OMM_EXT_LINKAGE_USAGE_COUNTS)` z komentarzem (`config_flag_decl.h:76-82`). Upstream **nigdy** nie ustawia `usageCounts` na linkage — potwierdzone negatywnie w całym #2519 (`usageCounts`, `pUsageCounts`, `usageCountsCount` nie występują tam ani razu). Zera z memsetu są tam wartością docelową, nie przeoczeniem.

**6. Odrzucanie `DXGI_FORMAT_R8_UINT`** — `opacity_micromap.c:615-628`. Upstream akceptował z `FIXME_ONCE`.

**7. Odroczenie postbuildu po budowie OMM** — `command.c:21739-21755` → `acceleration_structure.c:564-576`.

**8. Zaślepka SERIALIZATION** — `opacity_micromap.c:481-496`. Upstream wystawiał realne zapytanie mimo braku kopii serializującej.

**9. `copy_ext` biorące widok + dziedziczenie histogramu** — `opacity_micromap.c:825` + `vkd3d_opacity_micromap_inherit_usage_histogram_ext`.

**10. Odwrócona kolejność dispatchy** — `command.c:21577` przed `:21583`.

**11. Diagnostyka dzisiejszej sesji** (14 miejsc):
```
opacity_micromap.c:359, 660-675, 688-694, 699-705, 782
resource.c:5389-5398
va_map.c:472-480
command.c:21673
device.c:1457-1471   <- to wymusza OMM_EXT_LINKAGE_USAGE_COUNTS, więc w obecnym buildzie
                        publikacja histogramu jest AKTYWNA mimo projektu "domyślnie off"
```

**12. Pliki `.orig`** — `device.c.orig`, `vkd3d_private.h.orig`, `swapchain.c.orig` w `libs/vkd3d/`. Nie kod, ale przed commitem wyczyścić.

### Czego z tej listy NIE usuwać — mimo że nie ma odpowiednika u upstreamu

- **pętla przepinająca `pNext`** (`command.c:21475-21494`) — bez niej wskaźniki zwisają po realokacji tablicy linkage;
- **pętla przeliczająca `pUsageCounts`** (`command.c:21464-21472`) — to JEST upstreamowy niezmiennik I1, tylko przeniesiony;
- **`d3d12_command_list_discard_omm_build_info_ext`** (`command.c:21408`) — naprawa błędu upstreamu, nie odstępstwo semantyczne;
- **dwa typy widoku** — decyzja 1.3;
- **unie `vkd3d_omm_*`** — nośnik dwubackendowości;
- **`vkd3d_view_map_get_view`** — jedyny element serii przeniesiony wiernie;
- **blok w `dxil.c`** — nowsze API dxil-spirv;
- **brak `d3d12_command_list_reset_query`** — różnica wersji bazowej.

---

## 4. RYZYKA

| # | Ryzyko | Jak wykryć | Co zrobić |
|---|---|---|---|
| R1 | **Bezwarunkowe wpięcie z `micromap == VK_NULL_HANDLE`** (usunięcie odpięcia przy VA=0) okazuje się tym, co sterownik odrzuca. To jest cofnięcie naszej zmiany, więc jeśli hang wraca dokładnie tu — mamy odpowiedź. | Walidacja (`VKD3D_CONFIG=vk_debug,dxr12`) + device-lost przy **pierwszym** buildzie BLAS z geometrią OMM. Breadcrumb `build_rtas` bez następnika. | Jeśli hang wraca: to nie upstream jest źródłem błędu, tylko sterownik/nasza baza. Odnotować i zatrzymać bisekcję. |
| R2 | **Realne zapytanie SERIALIZATION** zwraca prawdziwy rozmiar, a `convert_copy_mode` nadal nie umie SERIALIZE → aplikacja alokuje bufor i dostaje śmieci. Upstream miał tę samą dziurę i się nią nie przejmował. | `FIXME("Unsupported OMM copy mode #%x.\n", mode)` w logu przy jednoczesnym braku błędu z postbuildu. Korupcja po stronie aplikacji, nie sterownika. | Punkt bisekcji: jeśli tytuł nie woła SERIALIZATION w ogóle (a Cyberpunk najpewniej nie), ryzyko jest zerowe. Sprawdzić logiem przed przywróceniem. |
| R3 | **`VK_INDEX_TYPE_UINT8`** narusza VUID, który nasz kod dziś cytuje (`VUID-...-indexType-10719`), albo wymaga `VK_KHR_index_type_uint8`/maintenance5. | Warstwa walidacyjna zgłosi VUID przy pierwszym BLAS z 8-bitowymi indeksami OMM. | Objawia się tylko na tytułach faktycznie używających 8-bitowych indeksów. Jeśli walidacja krzyczy — cofnąć **tylko ten** case, resztę zostawić. |
| R4 | **Zwisające `pNext`** gdyby ktoś przy „porządkach" usunął pętlę naprawczą jako „niewystępującą u upstreamu". | `VKD3D_CONFIG=fault,dxr12` + `EXT_device_address_binding_report`; wymusić partię z ≥2 buildami BLAS z geometrią OMM. Objaw: `VkAccelerationStructureTrianglesOpacityMicromapEXT` z losowym `sType`. | Pętla ma **zostać**. Opisać komentarzem, żeby nikt jej nie wyciął przy następnej rundzie. |
| R5 | **`dxr12` zdejmuje KHR** → tytuł, który działał na KHR, przestaje działać w trybie testowym, i wygląda to jak regresja EXT-a, a jest regresją konfiguracji. | Log: `INFO("DXR 1.2 support enabled.\n")` musi się pojawić, a `d3d12_device_uses_ext_opacity_micromap` musi być prawdą. Warto dodać jednorazowe `INFO` przy wyborze backendu. | Zawsze porównywać A/B na tej samej binarce, przełączając wyłącznie `VKD3D_CONFIG`. |
| R6 | **Współistnienie AS + OMM na jednym VA** nadal możliwe strukturalnie; naprawa z 2.3a zamyka znane drzwi, nie wszystkie. | Opcjonalna diagnostyka pod `VKD3D_CONFIG=fault`: w `vkd3d_va_map_place_acceleration_structure` sondować `try_read_opacity_micromap_ext` i zapalać jednorazowe `FIXME` — odpowiednik upstreamowego `FIXME("Attempted to place RTAS on VA #%"PRIx64" previously used by OMM.\n", va)`. | Dodać jako **diagnostykę**, nie jako część ścieżki. Jeśli się zapala — jest kolejne miejsce do naprawy. |
| R7 | **Rozjazd `usageCounts` między wymiarowaniem a budową** (niezmiennik I2) — po usunięciu histogramu z linkage powinien zniknąć, ale trzeba to potwierdzić. | Porównać `micromapSize` z `GetRaytracingAccelerationStructurePrebuildInfo` z faktycznym rozmiarem bufora docelowego przy buildzie. Log jednorazowy z obu miejsc. | Jeśli się rozjeżdża mimo usunięcia histogramu — winna jest jeszcze jakaś asymetria między `device.c` a `command.c`. |
| R8 | **Brak nagłówków Vulkana.** `VK_BUILD_ACCELERATION_STRUCTURE_ALLOW_OPACITY_MICROMAP_DATA_UPDATE_BIT_*` i `VK_INDEX_TYPE_UINT8` mogą wymagać bumpu; upstream bumpował `khronos/Vulkan-Headers` do `b39ab380a4` właśnie w #2519. | Kompilacja. | Sprawdzić przed pisaniem kodu, nie po. |
| R9 | **`d3d12_build_flags_to_vk` bez `device`** — dołożenie bitu DATA_UPDATE bez bramki to naruszenie VU. | Kompilacja przejdzie, walidacja nie. | Dodać parametr `device` i zbramkować (p. 2.3b). Nie iść na skróty. |
| R10 | **`vkd3d_memory_info_init` używa tieru** — jeśli ktoś „ujednolici" bramki i wstawi tam `supports_ray_tracing_tier_1_2`, `d3d12_caps` są jeszcze puste. | Cicha zmiana `memoryTypeBits` → alokacja trafia w typ pamięci, którego bufor nie przyjmie. Trudne do wykrycia. | `resource.c:11281` ma już komentarz ostrzegawczy. Nie ruszać tego miejsca. |
| R11 | **Kolizja nazw** `vkd3d_opacity_micromap_emit_immediate_postbuild_info` — nasza KHR-owa vs upstreamowa EXT-owa, identyczne nazwy, różne typy uchwytu. | Kompilator wyłapie tylko przy zmianie sygnatury; przy zgodnych typach 64-bitowych **nie wyłapie nic**. | Trzymać sufiks `_ext` bezwyjątkowo. Nie „upraszczać" nazw do upstreamowych. |

---

## 5. CZEGO NIE USTALONO

**5.1 PR #2505 (baza, draft, NIEscalony) — nigdy nie pobrany.** Cała flota pracowała na #2507/#2512/#2515/#2519. Jeśli gdziekolwiek jest ciało `vkd3d_va_map_place_opacity_micromap` w pierwotnej postaci albo pierwotne okablowanie flagi `dxr12`, to tam. Jest to draft, więc jego treść może się różnić od scalonego stanu — traktować jako źródło poszlak, nie prawdy.

**5.2 Pętla naprawiająca `pNext` dla `omm_infos` po realokacji — NIE ZNALEZIONA W ŻADNYM DIFFIE.** W #2519 jej nie ma; w #2507/#2512/#2515 też jej nie widziano. Upstreamowa struktura alokacji (`vkd3d_array_reserve` na `geometry_info_count + geometry_count` przed konwersją) ma tę samą podatność co nasza, a mimo to upstream nie ma odpowiednika naszego `fixup`. Trzy możliwości, żadnej nie da się rozstrzygnąć z posiadanych diffów: (a) upstream miał ten błąd i nikt na niego nie trafił; (b) pętla jest w scalonym drzewie czerwca 2025, wprowadzona poza tą czwórką PR-ów; (c) coś w upstreamowej kolejności alokacji czyni realokację niemożliwą. **Do sprawdzenia przez `git log`/`git show` na tagu z czerwca 2025 przed uznaniem naszego `fixup` za odstępstwo.**

**5.3 Czy `d3d12_command_list_reset_query` był w czerwcowej bazie wymagany.** Upstream go miał, nasza baza go nie ma **ani w ścieżce AS, ani w OMM**. Wniosek „różnica wersji bazowej" jest wnioskiem z symetrii, nie z odczytanej historii. Jeśli okaże się, że nasz `d3d12_command_allocator_allocate_query_from_type_index` **nie** resetuje slotu wewnętrznie, to jest to realny błąd w obu ścieżkach naraz — i wtedy sprawa wykracza poza OMM.

**5.4 Czy upstream kiedykolwiek bramkował promocję tieru flagą `dxr12`.** W #2519 promocja jest bezwarunkowa na bicie cechy. Cały plan opiera się na założeniu, że jedynym efektem flagi było odcięcie rozszerzenia na etapie rejestracji. Założenie jest spójne z #2507 (`VK_EXTENSION_COND`), ale nie zostało potwierdzone odczytem `d3d12_device_determine_ray_tracing_tier` z tamtej wersji.

**5.5 Białe znaki.** Wszystkie diffy pobierano przez WebFetch, który przepuszcza treść przez mały model. Struktura się spina (liczby hunków zweryfikowane niezależnymi zapytaniami; `opacity_micromap.c` deklaruje `@@ -0,0 +1,286 @@` i transkrypcja kończy się naturalnym końcem pliku), ale wierność co do spacji nie jest gwarantowana. Dwa znane miejsca z trailing whitespace: w `va_map.c` między `return;` a `if (view->info.buffer.rtas_is_micromap)` jest linia z czterema spacjami — w #2507 i ponownie w #2515. Przed commitem warto raz porównać surowy diff.

**5.6 Jedna odpowiedź pośrednika była niepełna** — wyszukiwanie `usageCountsCount` w #2512 zwróciło 1 trafienie zamiast 2, pomijając `build_info->usageCountsCount = desc->NumOmmHistogramEntries;` widoczne wprost w transkrypcji. Ufamy transkrypcji, nie wyszukiwarce. Pozostałe wyszukiwania były spójne.

**5.7 Nie sprawdzono w naszym drzewie przed pisaniem patcha:**
- czy `vkd3d_acceleration_structure_write_postbuild_info` liczy offset w sposób pozwalający na wariant z `desc_offset` (2.3a zakłada `0` i to trzeba potwierdzić odczytem `:442-450`);
- czy `has_incoming_uav` (`acceleration_structure.c:549-550`) czytające `.rtas` dla wpisów OMM ma gdziekolwiek skutek funkcjonalny;
- pełna treść `patches/vkd3d-proton/0005-vkd3d-ext-opacity-micromap-fallback.patch` (108106 B, 2160 linii) — porównania robiono wobec **drzewa po nałożeniu**, co jest wierniejsze, ale patch może zawierać komentarze wyjaśniające decyzje, których w drzewie nie widać.

**5.8 Czego cała seria #2505–#2519 nie zawiera i co po odtworzeniu nadal nie będzie działać:**
`CURRENT_SIZE` postbuild (Vulkan tego nie umie zapytać — upstreamowe TODO mówi o trzymaniu bufora obok), kopie `SERIALIZE`/`DESERIALIZE` mikromap (`vkCmdCopyMicromapToMemoryEXT` / `vkCmdCopyMemoryToMicromapEXT` nie występują w żadnym z czterech PR-ów), tryb `UPDATE` mikromapy (`mode` zawsze `VK_BUILD_MICROMAP_MODE_BUILD_EXT`, `UpdateScratchDataSizeInBytes` zawsze 0). To nie są luki do zapełnienia — to są granice odtwarzanej implementacji.

---

# Zestawienie z naszym kodem

# ZESTAWIENIE: upstream (#2507/#2512/#2515/#2519) vs nasze drzewo `/var/home/michael/.git/proton-11/vkd3d-proton/`

Wszystkie ścieżki bezwzględne. Dodatki diagnostyczne (POMIAR/DIAG-ADDR, wymuszenie FAULT + OMM_EXT_LINKAGE_USAGE_COUNTS) pominięte w porównaniu — ich spis na końcu.

---

## 0. USTALENIE NADRZĘDNE, KTÓRE ZMIENIA OBRAZ

**Flaga `dxr12` istnieje w drzewie, ale NIE JEST NIGDZIE UŻYWANA.**
```
/var/home/michael/.git/proton-11/vkd3d-proton/include/private/config_flag_decl.h:32
    VKD3D_DECL_CONFIG("dxr12", DXR_1_2)
```
`grep -rn "DXR_1_2"` po całym drzewie daje wyłącznie tę jedną linię deklaracji. Nie ma ani jednego `VKD3D_CONFIG_FLAG_IS_SET(DXR_1_2)` ani `VKD3D_CONFIG_FLAG_STATIC(DXR_1_2)`. Helper `d3d12_device_supports_ray_tracing_tier_1_2` **nie istnieje w naszym drzewie w ogóle** — upstream używał go jako jedynego punktu prawdy w 6 miejscach.

To znaczy: gniazdo na „osobną ścieżkę włączaną zmienną środowiskową" jest puste i gotowe, ale każde jedno miejsce, gdzie upstream pytał o tier 1.2, u nas pyta o coś innego (`d3d12_device_uses_ext_opacity_micromap` albo `supports_opacity_micromap`).

---

## 1. `libs/vkd3d/opacity_micromap.c` (856 linii vs 286+45 upstreamowych)

### Upstream robi, my nie

| upstream | nasz stan |
|---|---|
| `vkd3d_opacity_micromap_write_postbuild_info(list, desc, **desc_offset**, micromap)` — publiczna, z offsetem deskryptora | `vkd3d_opacity_micromap_write_postbuild_info_ext(list, desc, vk_micromap)` (:451) — **static**, bez `desc_offset`, `offset = desc->DestBuffer - resource->va` i koniec |
| `switch` po `InfoType` ustawiający trójkę (`vk_query_type`, `type_index`, `stride`), wyjście do wspólnego kodu | `if/else if/else`, obie niekompaktowe gałęzie mają własny `return`; `stride` to stała funkcji `const VkDeviceSize stride = sizeof(uint64_t);` |
| **REALNE** `VK_QUERY_TYPE_MICROMAP_SERIALIZATION_SIZE_EXT` + zerowanie tylko drugiego uint64 pod `offset + sizeof(uint64_t)` | ZAŚLEPKA (:481-496): `FIXME_ONCE(...)` + `vkCmdFillBuffer(..., offset, 2 * sizeof(uint64_t), 0)` — zeruje OBA pola, żadnego zapytania |
| `vkd3d_opacity_micromap_emit_postbuild_info(list, desc, count, addresses)` — wariant po tablicy VA, stride 2×8 dla SERIALIZATION | **nie istnieje**. (W #2512 była martwa, ale #2515 dołożył jej odpowiednik w AS — patrz p. 2) |
| `vkd3d_opacity_micromap_copy(list, dst, **VkMicromapEXT src_omm**, mode)`, bez memsetu, `info.mode` wypełniany dopiero w `if` | `vkd3d_opacity_micromap_copy_ext(list, dst, **const struct vkd3d_view *src_omm_view**, mode)` (:825), z memsetem, `convert_copy_mode_ext` sprawdzany PRZED placementem dst |
| kolejność: place dst → jeśli NULL to ERR → wypełnij info → `if(convert_copy_mode)` → `vkCmdCopyMicromapEXT` | odwrócona: `if (!convert_copy_mode_ext(...)) return;` → place dst → dziedziczenie histogramu → copy |
| `d3d12_build_flags_to_vk` **zwracające `VkBuildMicromapFlagsEXT`**, cicho ignorujące ALLOW_UPDATE | `d3d12_build_flags_to_vk_ext` (:48) — te same trzy bity, plus `FIXME_ONCE("ALLOW_UPDATE ... ignoring")`. Zachowanie równe, kształt inny |
| `d3d12_format_to_vk` zwracające `VkOpacityMicromapFormatEXT` | zwraca `VkOpacityMicromapFormatKHR` (:70) i jest **współdzielone przez oba backendy**. Aliasy, więc wartości te same |

### My robimy, upstream nie

1. **Cała warstwa histogramu usageCounts** — bez odpowiednika w ŻADNEJ upstreamowej części serii:
   - `vkd3d_opacity_micromap_publish_usage_histogram_node_ext` (:235) — push-only stos, CAS, cap `VKD3D_OMM_USAGE_HISTOGRAM_MAX_NODES_EXT 256u`, deduplikacja po `memcmp`
   - `vkd3d_opacity_micromap_publish_usage_histogram_ext` (:297)
   - `vkd3d_opacity_micromap_inherit_usage_histogram_ext` (:318) — dziedziczenie przy CLONE/COMPACT
   - `vkd3d_opacity_micromap_lookup_usage_histogram_ext` (:345) — za flagą `OMM_EXT_LINKAGE_USAGE_COUNTS`, domyślnie zwraca 0
2. **Podwójny konwerter linkage**: `..._convert_opacity_micromap` (KHR, :588) i `..._convert_opacity_micromap_ext` (:630).
3. **Podwójny konwerter typu indeksu** `..._index_type` (:560) / `..._index_type_ext` (:615); wersja EXT **odrzuca `DXGI_FORMAT_R8_UINT`** (`FIXME_ONCE` + `return false`), upstream #2519 go **akceptował**: `FIXME_ONCE("Using UINT8 as OMM index format is technically out of spec.\n"); omm->indexType = VK_INDEX_TYPE_UINT8;`
4. **Osobny przebieg rozwiązywania uchwytów**: `vkd3d_acceleration_structure_resolve_omm_va_maps` (:722, KHR) i `..._ext` (:756, z parametrem `bool best_effort`). Upstream nie ma tego w ogóle.
5. `vkd3d_opacity_micromap_emit_immediate_postbuild_info` (:414) o **sygnaturze KHR-owej** (`VkAccelerationStructureKHR`, dodatkowy `VkDeviceAddress va`), delegujące do wspólnego `vkd3d_acceleration_structure_write_postbuild_info(..., VKD3D_RTAS_KIND_NON_TLAS)`. Kolizja nazw z upstreamową funkcją o identycznej nazwie, ale innym typie — pułapka przy przepisywaniu.

### Zachowanie równe, kształt inny

- `vkd3d_opacity_micromap_end_barrier` (:393) — **bajt w bajt** jak upstream, łącznie z komentarzem.
- Bariera w `..._emit_immediate_postbuild_info_ext` (:540-545) — **dokładnie te same bity** (`MICROMAP_WRITE` → `MICROMAP_BUILD|COPY` / `MICROMAP_READ|TRANSFER_WRITE`), inny komentarz + dodatkowy `RT_TRACE`.
- `vkd3d_opacity_micromap_convert_inputs_ext` (:195) — **semantycznie identyczne** z upstreamowym `convert_inputs`: memset, sType, type, flags, `mode = VK_BUILD_MICROMAP_MODE_BUILD_EXT`, `usageCountsCount`, `pUsageCounts`, `data.deviceAddress`, `triangleArray.deviceAddress`, `triangleArrayStride`, `return true` bezwarunkowo. Różnica kształtu: pętla po histogramie wyniesiona do `..._convert_inputs_usages_ext` (:169), która zwraca licznik.
- Brak `d3d12_command_list_reset_query` przed `vkCmdWriteMicromapsPropertiesEXT` — **to NIE jest nasze odstępstwo w ścieżce OMM**: `vkd3d_acceleration_structure_write_postbuild_info` (`acceleration_structure.c:442-450`) też go nie ma. Nasza baza jest nowsza niż czerwiec 2025 i reset przeniósł się gdzie indziej. Przy odtwarzaniu 1:1 **nie dodawać** resetu na siłę.

---

## 2. `libs/vkd3d/acceleration_structure.c`

### Upstream robi, my nie — TO JEST NAJPOWAŻNIEJSZA LUKA

**Hunk `@@ -408,9 +409,28 @@` z #2515 (rozgałęzienie AS/OMM w postbuild info) NIE ZOSTAŁ ODTWORZONY W ŻADNEJ FORMIE.**

Nasze `vkd3d_acceleration_structure_emit_postbuild_info` (:515) jest `static`, bierze **jeden** `rtas_va` i wygląda tak:
```c
    vkd3d_va_map_try_read_rtas(&list->device->memory_allocator.va_map, list->device, rtas_va,
            &vk_acceleration_structure, &rtas_kind);

    if (vk_acceleration_structure == VK_NULL_HANDLE)
    {
        WARN("Emit postbuild placing unknown AS at #%" PRIx64 " future BLP queries may not be reliable.\n", rtas_va);
        rtas_kind = VKD3D_RTAS_KIND_UNKNOWN;
        vk_acceleration_structure =
                vkd3d_va_map_place_acceleration_structure(&list->device->memory_allocator.va_map, list->device,
                rtas_va, rtas_kind);
    }
```
Zero gałęzi mikromapy. `vkd3d_va_map_try_read_rtas` szuka **wyłącznie** klucza `VKD3D_VIEW_TYPE_ACCELERATION_STRUCTURE`.

**Skutek, którego upstream nie mógł mieć:** aplikacja woła `EmitRaytracingAccelerationStructurePostbuildInfo` na VA tablicy OMM → nasz kod idzie tą ścieżką → `try_read_rtas` chybia (bo pod tym VA jest widok typu `OPACITY_MICROMAP_EXT`, nie `ACCELERATION_STRUCTURE`) → `place_acceleration_structure` **TWORZY VkAccelerationStructureKHR na tym samym zakresie bufora, na którym już siedzi VkMicromapEXT** → `vkCmdWriteAccelerationStructuresPropertiesKHR` na strukturze, która nigdy nie była budowana. Jedyny ślad to `WARN`.

U upstreamu to **niemożliwe z konstrukcji** — jeden typ widoku (`ACCELERATION_STRUCTURE_OR_OPACITY_MICROMAP`) plus `bool rtas_is_micromap` sprawiają, że pod jednym VA żyje ALBO AS ALBO OMM, i #2515 jawnie rozgałęział na `vkd3d_opacity_micromap_write_postbuild_info`. To jest bezpośrednia konsekwencja naszej decyzji o dwóch typach widoku i moim zdaniem najlepszy nowy kandydat na przyczynę zawieszenia.

Drugie odstępstwo w tym pliku: **`ALLOW_OMM_LINKAGE_UPDATE` mapujemy na JEDEN bit**, upstream #2519 na dwa:
```c
/* nasz, acceleration_structure.c:39-43 */
    if (flags & D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_ALLOW_OMM_LINKAGE_UPDATE)
    {
        /* D3D12 spec isn't clear on what is allowed to be updated and when */
        vk_flags |= VK_BUILD_ACCELERATION_STRUCTURE_ALLOW_OPACITY_MICROMAP_UPDATE_BIT_KHR;
    }
```
upstream dokładał jeszcze `VK_BUILD_ACCELERATION_STRUCTURE_ALLOW_OPACITY_MICROMAP_DATA_UPDATE_BIT_EXT`.

### My robimy, upstream nie

- `vkd3d_acceleration_structure_flush_postbuild_batch` (:545) — **cały mechanizm batchowania postbuildu**, z trójdrożnym rozgałęzieniem (generic query / `is_omm` / immediate AS) i podwójnym gniazdem KHR/EXT (:566-575). Upstream emitował natychmiast.
- `vkd3d_acceleration_structure_begin_query_barrier` (:325) z dołożonymi bitami MICROMAP przy `d3d12_device_uses_ext_opacity_micromap` (:347-353).
- `vkd3d_acceleration_structure_convert_inputs` bierze `const union vkd3d_omm_triangles_info *` zamiast gołego `VkAccelerationStructureTrianglesOpacityMicromapEXT *`, i memsetuje **warunkowo, per backend** (:165-171 dla TLAS, :203-209 dla BLAS). Upstream miał jeden bezwarunkowy `memset(omm_infos, ...)` tylko w gałęzi BLAS.
- Bramka w `case D3D12_RAYTRACING_GEOMETRY_TYPE_OMM_TRIANGLES`: `if (!device->device_info.supports_opacity_micromap)` (:276) zamiast `d3d12_device_supports_ray_tracing_tier_1_2(device)`.
- `vkd3d_acceleration_structure_copy` ma piąty parametr `enum vkd3d_rtas_kind rtas_kind` (:608).

### Zachowanie równe, kształt inny

- `vkd3d_acceleration_structure_get_geometry_count` (:62) — `if (desc->Type != ..._BOTTOM_LEVEL) return 1;` — **identyczne** z #2519.
- `vkd3d_acceleration_structure_convert_triangles` (:71) — **znak w znak identyczne** z #2519, jedyna różnica: `WARN(...)` → `TRACE("Application is using IndexBuffer = 0 ...")` (:85).
- Kolejność w `case OMM_TRIANGLES`: mieszanka typów → bramka → `convert_triangles` → konwersja linkage. Ta sama co upstream, tylko rozgałęziona per backend (:285-297).

---

## 3. `libs/vkd3d/command.c`

### Upstream robi, my nie

1. **Natychmiastowy postbuild po budowie OMM.** Upstream:
   ```c
   if (num_postbuild_info_descs)
   {
       d3d12_command_list_flush_rtas_batch(list);
       vkd3d_opacity_micromap_emit_immediate_postbuild_info(list,
               num_postbuild_info_descs, postbuild_info_descs, build_info->dstMicromap);
   }
   ```
   My (`command.c:21739-21755`) **kolejkujemy**:
   ```c
       info->desc = postbuild_info_descs[i];
       info->rtas_vk.micromap_ext = micromap;
       info->rtas_va = 0;
       info->is_omm = true;
   ```
   `grep -n "emit_immediate_postbuild_info" command.c` → **zero trafień**. Rozwiązanie następuje dopiero w `acceleration_structure.c:568`.
2. **Rozgałęzienie AS/OMM w `CopyRaytracingAccelerationStructure` jednym `try_read`.** Upstream: jeden `if (tier_1_2)` → jedno `vkd3d_va_map_try_read_rtas(..., &src_as, &src_omm)` → trójdrożnie. My (`:22214-22258`): **dwie odrębne próby** — najpierw `if (d3d12_device_uses_ext_opacity_micromap)` + `try_read_opacity_micromap_view_ext` z ewentualnym `return`, potem bezwarunkowe `try_read_rtas`, potem placement z `WARN`.
3. **Rozgałęzienie w `EmitRaytracingAccelerationStructurePostbuildInfo`** — u nas go NIE MA (`:22167-22176` ustawia `info->is_omm = false` dla każdego VA). Patrz p. 2.
4. `d3d12_command_list_allocate_rtas_build_info` gatuje tablicę linkage na tierze 1.2; u nas na `list->device->device_info.supports_opacity_micromap` (`:21280`).

### My robimy, upstream nie

- **`d3d12_command_list_fixup_rtas_batch` (:21428)** — cały wydzielony etap naprawczy. W nim:
  - pętla przeliczająca `pUsageCounts` dla `.ext` (:21464-21472) — **to jest niezmiennik I1 upstreamu, odtworzony poprawnie**, tylko przeniesiony z `flush` do `fixup`;
  - **pętla przepinająca `pNext` linkage BLAS po realokacji (:21475-21494)** z sentinelem po `sType`. Upstream #2519 tego nie ma i nie potrzebuje, bo tam tablica `omm_infos` jest rezerwowana raz na `geometry_count` przed konwersją. U nas jest to konieczne, bo `vk_prepend_struct` w konwerterze zapisuje adres do tablicy, którą kolejne `vkd3d_array_reserve` może przenieść;
  - zdublowana wersja tego wszystkiego dla `.khr` (:21496-21547).
- **`d3d12_command_list_discard_omm_build_info_ext` (:21408)** — cofa liczniki na trzech ścieżkach błędu (:21660, :21669, :21679). Upstream przy błędzie po alokacji po prostu `return`, zostawiając w partii wpis częściowo wypełniony.
- **`d3d12_command_list_allocate_omm_build_info_ext` (:21375)** — rezerwuje TYLKO `omm_build_infos.ext` i `omm_usage_infos.ext`, nie dotykając `build_infos`/`geometry_infos`/`range_infos`. To jest zgodne z upstreamem #2512. Osobna KHR-owa `..._allocate_omm_build_info` (:21316) rezerwuje pięć tablic — to nasz dorobek.
- **Twardy błąd przy `DestAccelerationStructureData == 0`** (`:21664-21671`): `ERR("OMM array build without a destination address. Dropping call.\n")` + discard. Upstream: `if (desc->DestAccelerationStructureData) { ... }`, przy zerze `dstMicromap` zostaje 0 z memsetu i build **i tak leci**.
- **Publikacja histogramu** (`:21689`): `vkd3d_opacity_micromap_publish_usage_histogram_ext(micromap_view, desc->Inputs.pOpacityMicromapArrayDesc);`
- **Odwrócona kolejność dispatchy we `flush_rtas_batch`**: u nas `vkCmdBuildMicromapsEXT` (:21577) **PRZED** `vkCmdBuildAccelerationStructuresKHR` (:21583). Upstream #2512: AS pierwszy, mikromapy drugie. Nasz komentarz (:21569-21574) tłumaczy, że partia nigdy nie miesza obu, więc kolejność jest nieobserwowalna — ale to odstępstwo od litery.
- **Dodatkowa bramka backendu** przy dispatchu: `if (rtas_batch->omm_build_info_count && d3d12_device_uses_ext_opacity_micromap(list->device))`. Upstream: sam licznik.
- `d3d12_command_list_flush_rtas_barrier` (:21609) z dołożonymi bitami MICROMAP (:21625-21630) — u upstreamu #2512 odpowiednik był inline w `BuildRaytracingAccelerationStructure` i gatowany na `list->device->device_info.opacity_micromap_features.micromap`.
- `assume_hazard` (:22105-22110) uwzględniające `omm_build_info_count` — upstream miał `(rtas_batch->build_info_count || rtas_batch->omm_build_info_count) && rtas_batch->build_type != desc->Inputs.Type`, czyli **to samo**. Zgodne.
- Rejestracja zakresów scratcha (`d3d12_command_list_register_rtas_scratch_range`, :21968) — nasza baza, nie ma związku z OMM.

### Zachowanie równe, kształt inny

- Blok breadcrumbów OMM (:21693-21737) — **zgodny z upstreamem co do znaku**: te same tagi, ta sama kolejność AUX64/AUX32, ta sama pętla po histogramie, to samo `vkGetMicromapBuildSizesEXT` z `VK_ACCELERATION_STRUCTURE_BUILD_TYPE_DEVICE_KHR`. Drobiazgi: `VKD3D_CONFIG_FLAG_IS_SET(BREADCRUMBS)` zamiast `vkd3d_config_flags & VKD3D_CONFIG_FLAG_BREADCRUMBS`, `uint32_t i` zamiast `unsigned int i`, dodatkowa pusta linia.
- Trzy wstawki barierowe w `vk_access_and_stage_flags_from_d3d12_resource_state` (:6465-6471, :6484-6490, :6518-6520) — **te same bity, ta sama kolejność, te same trzy ramiona** co #2507. Różnica wyłącznie w bramce (`d3d12_device_uses_ext_opacity_micromap` vs `d3d12_device_supports_ray_tracing_tier_1_2`).
- Uzyskanie uchwytu: upstream `build_info->dstMicromap = vkd3d_va_map_place_opacity_micromap(...)` wprost; my (`:21674-21684`) przez widok, bo widok niesie histogram. Wynikowy `dstMicromap` jest ten sam.

---

## 4. `libs/vkd3d/va_map.c`

### Upstream robi, my nie

- **`vkd3d_va_map_try_place_rtas(va_map, device, va, bool rtas_is_omm, VkAccelerationStructureKHR *, VkMicromapEXT *)`** — jedna statyczna funkcja z dwoma wyjściami plus dwa cienkie opakowania. U nas nie istnieje.
- **Oba `FIXME` o kolizji VA**:
  `FIXME("Attempted to place RTAS on VA #%"PRIx64" previously used by OMM.\n", va)` i wersja odwrotna. U nas ich nie ma, bo kolizja jest z definicji niemożliwa (dwa typy widoku).
- **`vkd3d_va_map_try_read_rtas` z parametrem `VkMicromapEXT *micromap`** (#2515). Nasza wersja (:250) ma zamiast tego `enum vkd3d_rtas_kind *rtas_kind` i **nie umie zwrócić mikromapy w ogóle**.

### My robimy, upstream nie

- **Cztery funkcje zamiast dwóch**:
  - `vkd3d_va_map_try_read_opacity_micromap_view_ext` (:287) — zwraca `const struct vkd3d_view *`
  - `vkd3d_va_map_try_read_opacity_micromap_ext` (:314) — cienkie opakowanie na uchwyt
  - `vkd3d_va_map_place_opacity_micromap_view_ext` (:429)
  - `vkd3d_va_map_place_opacity_micromap_ext` (:489)
- **Maszyna stanów `rtas_kind`** — pętla CAS UNKNOWN/TLAS/NON_TLAS/MUTATED w `vkd3d_va_map_place_acceleration_structure` (:392-424) plus `vkd3d_get_rtas_kind_string` (:326). Zero śladu u upstreamu. W `place_opacity_micromap_view_ext` jej celowo nie ma (:484-486).
- **`key.u.buffer.usage = 0`** we wszystkich czterech miejscach (:276, :307, :386, :482). Pole nie istniało w czerwcowym drzewie i jest hashowane (`resource.c:1386`).

### Zachowanie równe, kształt inny

- **Idiom rozmiaru jest IDENTYCZNY.** `key.u.buffer.size = resource->size - key.u.buffer.offset;` (:471) — dokładnie to samo, co upstream robił dla OMM. Hipoteza z POMIARU 3 nie znajduje potwierdzenia w upstreamie.
- `key.u.buffer.buffer/offset/format` — identyczne.
- Cały blok CAS-owania `view_map` (malloc → `vkd3d_view_map_init` → `vkd3d_atomic_ptr_compare_exchange` → sprzątanie po przegranej) — **skopiowany znak w znak**, łącznie z komentarzami, w obu naszych funkcjach `place_*`.
- `vkd3d_view_map_get_view` używane w obu `try_read_*` — spełnia niezmiennik #2515 („odczyt nigdy nie tworzy obiektu"). Zgodne.

---

## 5. `libs/vkd3d/resource.c`

### Upstream robi, my nie

- `vkd3d_create_opacity_micromap_view` wypełniało `VkMicromapCreateInfoEXT` **polem po polu, bez memsetu**, w tym jawnie `create_info.createFlags = 0;` i `create_info.deviceAddress = 0;`.
- Upstream kończył ustawieniem `object->info.buffer.rtas_is_micromap = true;` i **resetował to pole na `false` w `vkd3d_create_buffer_view` i `vkd3d_create_acceleration_structure_view`** (bo `vkd3d_view_create` nie zeruje struktury).
- Wspólny `case VKD3D_VIEW_TYPE_ACCELERATION_STRUCTURE_OR_OPACITY_MICROMAP` w hashu, porównaniu klucza, `create_view2` i `vkd3d_view_destroy`, rozgałęziany po `rtas_is_micromap`.

### My robimy, upstream nie

- **Osobny `VKD3D_VIEW_TYPE_OPACITY_MICROMAP_EXT`** wszędzie: hash (:1380), compare (:1449), `create_view2` (:1611-1614), destroy (:4970-4985).
- `vkd3d_create_opacity_micromap_view_ext` (:5369) — z memsetem, 4 przypisania + typ, plus:
  ```c
  object->info.buffer.rtas_kind = VKD3D_RTAS_KIND_UNKNOWN; /* Pre-publish, no atomic store required */
  object->info.buffer.omm_histogram_ext = NULL;
  ```
- **Zwalnianie listy histogramu w `vkd3d_view_destroy`** (:4975-4981) — pętla po `omm_histogram_ext` przed `vkDestroyMicromapEXT`.
- `vkd3d_view_map_create_view2` bierze `enum vkd3d_rtas_kind rtas_kind` zamiast `bool rtas_is_omm` (:1580-1582).

### Zachowanie równe, kształt inny

- **`vkd3d_view_map_get_view` (:1563-1578) jest ZNAK W ZNAK identyczne z #2515**, łącznie z komentarzem o read-write spinlockach. Jedyny element całej serii przeniesiony wiernie.
- Bity usage w `vkd3d_create_buffer` (:235-243): `MICROMAP_STORAGE` tylko dla `heap_type == DEFAULT || !is_cpu_accessible_heap`, `MICROMAP_BUILD_INPUT_READ_ONLY` zawsze — **układ identyczny** z #2507, różni się tylko bramka (`d3d12_device_uses_ext_opacity_micromap` vs `d3d12_device_supports_ray_tracing_tier_1_2`).
- Sonda w `vkd3d_memory_info_init` (:11286-11293): dodaje oba bity. Nasza bramka to `accelerationStructure && rayTracingPipeline && uses_ext_omm` — to **dokładnie lustro** gatunku z `vkd3d_create_buffer` minus odczyt `d3d12_caps` (`d3d12_device_supports_ray_tracing_tier_1_0` czyta `d3d12_caps.options5.RaytracingTier`, a `vkd3d_memory_info_init` biegnie wcześniej). Niezmiennik „sonda ⊇ realne bufory" utrzymany. Poprawnie.
- `VkMicromapCreateInfoEXT`: memset + `sType/buffer/offset/size/type`. Funkcjonalnie równoważne upstreamowi (`createFlags = 0`, `deviceAddress = 0` z memsetu).

---

## 6. `libs/vkd3d/device.c`

### Upstream robi, my nie

| upstream | nasz stan |
|---|---|
| `VK_EXTENSION_COND(EXT_OPACITY_MICROMAP, EXT_opacity_micromap, VKD3D_CONFIG_FLAG_DXR_1_2)` — **opt-in, domyślnie WYŁĄCZONE** | `device.c:98-99`: `VK_EXTENSION_DISABLE_COND(KHR_OPACITY_MICROMAP, ..., NO_DXR)` i `VK_EXTENSION_DISABLE_COND(EXT_OPACITY_MICROMAP, ..., NO_DXR)` — **opt-out, domyślnie WŁĄCZONE** |
| `{"dxr12", VKD3D_CONFIG_FLAG_DXR_1_2}` w `vkd3d_config_options[]` | flaga zadeklarowana w `config_flag_decl.h:32`, **nigdzie nieużywana** |
| `d3d12_device_supports_ray_tracing_tier_1_2()` — koniunkcja bitu Vulkana I zadeklarowanego tieru D3D12 | **funkcja nie istnieje** |
| `if (tier == D3D12_RAYTRACING_TIER_1_1 && info->opacity_micromap_features.micromap)` | `device.c:10250`: `if (tier == D3D12_RAYTRACING_TIER_1_1 && info->supports_opacity_micromap)` — **promocja bezwarunkowa**, bez flagi |
| jedna struktura feature | dwie (`opacity_micromap_features` KHR :5493, `opacity_micromap_features_ext` :5495) |

### My robimy, upstream nie

- **Sonda wyboru backendu** (`:2164-2192`) — osobne, jednorazowe `vkGetPhysicalDeviceFeatures2` na lokalnym łańcuchu, żeby wyłączyć EXT, gdy KHR jest użyteczny. Plus rozstrzygnięcie (`:2802-2806`):
  ```c
  info->using_khr_opacity_micromap = info->opacity_micromap_features.micromap;
  if (info->using_khr_opacity_micromap)
      info->opacity_micromap_features_ext.micromap = VK_FALSE;
  info->supports_opacity_micromap = info->using_khr_opacity_micromap ||
          info->opacity_micromap_features_ext.micromap;
  ```
- **Dwutorowe prebuild info**: `d3d12_device_get_raytracing_opacity_micromap_array_prebuild_info` (:8998) sprawdza `supports_opacity_micromap`, po czym deleguje do `..._ext` (:8954) albo idzie KHR-em.
- **Wywołanie `resolve_omm_va_maps_ext` w ścieżce wymiarowania BLAS** (`:9132-9136`), z `best_effort = true`. Upstream nie ma tego w ogóle — u niego konwerter rozwiązywał uchwyt raz i to samo widziały obie ścieżki.
- **Unia stosowa** na tablicę linkage (`:9066-9070`) i podwójne `vkd3d_malloc` per backend (:9110-9113).
- Komentarz przy puli `OMM_SERIALIZE_SIZE` (:4721-4723) o tym, że jest nieosiągalna.

### Zachowanie równe, kształt inny

- `d3d12_device_get_raytracing_opacity_micromap_array_prebuild_info_ext` (:8954-8996) — **niemal znak w znak** jak upstreamowa `d3d12_device_get_raytracing_opacity_micromap_array_prebuild_info` z #2512: `usages_stack[VKD3D_BUILD_INFO_STACK_COUNT]`, malloc powyżej progu, `convert_inputs_ext`, `VK_STRUCTURE_TYPE_MICROMAP_BUILD_SIZES_INFO_EXT`, `vkGetMicromapBuildSizesEXT` z `VK_ACCELERATION_STRUCTURE_BUILD_TYPE_DEVICE_KHR`, `micromapSize` → `ResultDataMaxSizeInBytes`, `buildScratchSize` → `ScratchDataSizeInBytes`, `UpdateScratchDataSizeInBytes = 0`, etykieta `cleanup:`. Jedyna różnica: bramkę na wsparcie przeniesiono do funkcji-rodzica.
- Pule zapytań (:4715-4726) — indeksy `7u/8u/9u` i oba `queryType` **identyczne** z #2512.
- `vkd3d_init_shader_extensions` (:11442-11445) — publikuje `VKD3D_SHADER_TARGET_EXTENSION_OPACITY_MICROMAP`, gate `supports_opacity_micromap` zamiast `opacity_micromap_features.micromap` (czyli publikuje też dla KHR).

---

## 7. `libs/vkd3d/vkd3d_private.h`

**Upstream / my** — tabela struktur:

| upstream | my |
|---|---|
| `VKD3D_VIEW_TYPE_ACCELERATION_STRUCTURE_OR_OPACITY_MICROMAP` (przemianowanie) | `VKD3D_VIEW_TYPE_ACCELERATION_STRUCTURE` + osobny `VKD3D_VIEW_TYPE_OPACITY_MICROMAP_EXT` (:1376-1380) |
| `bool rtas_is_micromap; /* not hashed */` w `info.buffer` | `uint32_t rtas_kind; /* not hashed; accessed atomically */` + `struct vkd3d_omm_usage_histogram_ext *omm_histogram_ext;` (:1404-1409) |
| `VkMicromapBuildInfoEXT *omm_build_infos; size_t omm_build_info_count/size;` — **płaskie** | `union vkd3d_omm_build_info omm_build_infos;` (:3260) — unia khr/ext |
| `VkMicromapUsageEXT *omm_usage_infos;` — płaskie | `union vkd3d_omm_usage_info omm_usage_infos;` (:3264) |
| `VkAccelerationStructureTrianglesOpacityMicromapEXT *omm_infos; size_t omm_info_size;` | `union vkd3d_omm_triangles_info omm_triangles_infos; size_t omm_triangles_info_size;` (:3251-3252) |
| brak | `struct vk_acceleration_structure_postbuild_info` (:3221-3235) z unią `rtas_vk{rtas, micromap_ext}`, `bool is_omm`, `rtas_kind` — **cała warstwa odraczająca** |
| brak | `struct vkd3d_omm_usage_histogram_ext` (:386-391) z elastyczną tablicą |
| brak | `d3d12_device_uses_ext_opacity_micromap()` inline (:6351-6355) |
| `vkd3d_view_map_create_view2(..., bool rtas_is_omm)` | `vkd3d_view_map_create_view2(..., enum vkd3d_rtas_kind rtas_kind)` (:7111) |
| `vkd3d_va_map_try_read_rtas(..., VkAccelerationStructureKHR *, VkMicromapEXT *)` | `vkd3d_va_map_try_read_rtas(..., VkAccelerationStructureKHR *, enum vkd3d_rtas_kind *)` (:446-449) |

**Zgodne co do znaku:** indeksy zapytań `VKD3D_QUERY_TYPE_INDEX_OMM_COMPACTED_SIZE (7u)` / `OMM_SERIALIZE_SIZE (8u)` / `VKD3D_VIRTUAL_QUERY_TYPE_COUNT (9u)` (:2779-2781); breadcrumby `BUILD_OMM` (:4231) tuż po `BUILD_RTAS` i `COPY_OMM` (:4233) tuż po `COPY_RTAS`; `static inline vkd3d_view_map_create_view` jako opakowanie.

---

## 8. Pliki pozostałe

- **`libs/vkd3d/vulkan_procs.h` (:227-232)** — **DOKŁADNIE te same 6 PFN-ów** co #2507, w tej samej kolejności. Jedyna różnica: nasz komentarz nagłówkowy mówi wprost, że to fallback i że serializacja do pamięci nie jest ładowana. **Zero pracy do zrobienia.**
- **`libs/vkd3d/breadcrumbs.c` (:76, :78)** — `"build_omm"` i `"copy_omm"`, zgodne z #2512/#2515.
- **`libs/vkd3d/meson.build:91`** — `'opacity_micromap.c'` obecne, zgodne.
- **`libs/vkd3d/raytracing_pipeline.c`** — upstream #2519: `pipeline_create_info.flags |= VK_PIPELINE_CREATE_RAY_TRACING_OPACITY_MICROMAP_BIT_EXT;`. My (:2551-2552): `flags2.flags |= VK_PIPELINE_CREATE_2_RAY_TRACING_OPACITY_MICROMAP_BIT_KHR;` plus OR-owanie globala z `device_vkd3d_ext.c` (:1986-1987) i ustawianie `VKD3D_SHADER_INTERFACE_RAYTRACING_OPACITY_MICROMAP` (:2040). Bity są aliasami — zachowanie równe, kształt inny (flags2 zamiast flags).
- **`libs/vkd3d/device_vkd3d_ext.c`** — `D3D12_VK_EXT_OPACITY_MICROMAP` → `device->device_info.supports_opacity_micromap` (:151); globalny bit RT OMM (`:457-458`, `:472-473`) i `VerifyOpacityMicromapArrayNVAPI` w `command_list_vkd3d_ext.c:126`. **Warstwa NVAPI, której cała seria #2505–#2519 nie zawiera w ogóle.**
- **`libs/vkd3d-shader/dxil.c` (:1160-1174) — przekazanie do dxil-spirv.** Upstream #2507 miał cały blok w `#if 0` (dxil-spirv nie miał wtedy `DXIL_SPV_OPTION_OPACITY_MICROMAP`). U nas blok jest **AKTYWNY** i bogatszy niż upstreamowy stub:
  ```c
  dxil_spv_option_opacity_micromap helper = { { DXIL_SPV_OPTION_OPACITY_MICROMAP } };
  helper.trace_ray_enabled =
          (shader_interface_info->flags & VKD3D_SHADER_INTERFACE_RAYTRACING_OPACITY_MICROMAP)
                  ? DXIL_SPV_TRUE : DXIL_SPV_FALSE;
  helper.ray_query_force_omm_execution_mode_in_legacy_sm =
          (shader_interface_info->flags & VKD3D_SHADER_INTERFACE_RAY_QUERY_OMM_DEVICE_GLOBAL)
                  ? DXIL_SPV_TRUE : DXIL_SPV_FALSE;
  ```
  Upstreamowy stub ustawiał tylko `DXIL_SPV_TRUE` na jedynym polu. **Nie cofać tego do `#if 0`** — to nowsze API dxil-spirv i nasza baza go wymaga. Ścieżka shaderowa jest **wspólna dla obu backendów** (gate `supports_opacity_micromap` w `device.c:11442`), więc odtwarzana ścieżka EXT nie musi jej dotykać.

---

## 9. PODSUMOWANIA PRZEKROJOWE (tematy wskazane w zleceniu)

### Rozwiązywanie uchwytów
Upstream: **jeden punkt, natychmiast, zawsze** — `vkd3d_va_map_place_opacity_micromap()` wołane w konwerterze linkage (#2519), w build path (#2512) i w copy (#2515). Zero odroczeń, zero drugiego przebiegu.

My: **dwa punkty, konkurencyjne**. Konwerter rozwiązuje na miejscu (`opacity_micromap.c:695`, dorobek POMIARU 11), a zaraz potem `vkd3d_acceleration_structure_resolve_omm_va_maps_ext` **nadpisuje to samo pole** — z `command.c:21906` (`best_effort=false`, twardo) i z `device.c:9134` (`best_effort=true`, miękko). Odroczenie jako takie zostało już zdjęte, ale **asymetria polityki błędu między wymiarowaniem a budową pozostała** i jest wprost zabroniona przez upstreamowy niezmiennik N8/I2. Do tego postbuild jest u nas odroczony przez kolejkę `postbuild_infos` (upstream: natychmiast).

### usageCounts
- **Budowa tablicy OMM**: `usageCountsCount` + `pUsageCounts` ustawiane w konwerterze (`opacity_micromap.c:212-213`), przeliczane po realokacji we `fixup` (`command.c:21469`). **Zgodne z upstreamem, niezmienniki I1 i I2 utrzymane.**
- **Linkage BLAS**: upstream **nigdy** nie ustawia tam usageCounts (memset zostawia zera; potwierdzone negatywnie w całym #2519). My publikujemy je z zapamiętanego histogramu (`opacity_micromap.c:684-686`), za flagą `omm_ext_linkage_usage_counts`, domyślnie 0. **To konstrukcja wyłącznie nasza, do usunięcia przy odtwarzaniu 1:1.**

### `OpacityMicromapArray == 0`
- upstream #2519: doczepia strukturę **bezwarunkowo**, wypełnia `indexType`/`indexBuffer`/`indexStride`/`baseTriangle`, `micromap` zostaje `VK_NULL_HANDLE` z memsetu, jedzie dalej.
- my (`opacity_micromap.c:641-646`): **odpinamy całe wpięcie**, `memset(omm_triangles_info, 0, ...)` + `return true`, `pNext` w ogóle nie zostaje ustawione. Dodatkowo sprawdzamy `pOmmLinkage` na NULL, czego upstream nie robi.
- Konsekwencja: sterownik dostaje **inną geometrię** niż u upstreamu. Odtworzenie wymaga usunięcia tego bloku.
- Trzecie odstępstwo: przy `VA != 0` i nieudanym placement upstream tylko loguje `ERR`; my `return false` (`opacity_micromap.c:706-711`) → propaguje na `"Failed to convert inputs."` → **cała budowa RTAS porzucona**.

### Bariery
- Trzy wstawki w `vk_access_and_stage_flags_from_d3d12_resource_state`: **bit w bit zgodne** z #2507, inna bramka.
- `vkd3d_opacity_micromap_end_barrier`: **identyczna**.
- Bariera wejściowa w `emit_immediate_postbuild_info_ext`: **identyczne bity**.
- **Nadmiar u nas**: `flush_rtas_barrier` (:21625-21630), `begin_query_barrier` (`acceleration_structure.c:347-353`), oraz **podwojenie** — `flush_postbuild_batch` wystawia `begin_query_barrier`, a wewnątrz każdy element OMM wystawia własną barierę wejściową + `end_barrier`, po czym na końcu leci `end_query_barrier`. Upstream miał dokładnie jedną parę na wywołanie.
- **Brak u nas**: bariery przy zmianie `build_type` z warunkowym `if (opacity_micromap_features.micromap)` dokładającym parę `MICROMAP_READ|WRITE` do src i dst — u nas to `flush_rtas_barrier` z innym zestawem (dodaje `MICROMAP_WRITE` do src i `MICROMAP_READ` do dst, ale nie obu do obu).

### Batching
- Alokacja partii OMM EXT (`allocate_omm_build_info_ext`): zgodna z upstreamem (dwie tablice, dwa liczniki).
- Przeliczenie `pUsageCounts`: zgodne, przeniesione do `fixup`.
- Kolejność dispatchy: **odwrócona** (u nas OMM przed AS).
- **Naprawa `pNext` linkage po realokacji** (:21475-21494): nasza, konieczna przy naszym kształcie alokacji, u upstreamu zbędna.
- `discard` przy błędzie: nasza, ostrożniejsza niż upstream.
- Postbuild: u nas **osobna kolejka** `postbuild_infos` flushowana razem z partią; u upstreamu natychmiast.

### Czas życia obiektów
- upstream: jeden VA = jeden rodzaj obiektu, na zawsze, kolizja wykrywana `FIXME`.
- my: **jeden VA może jednocześnie nieść `VkAccelerationStructureKHR` i `VkMicromapEXT`**, bo klucze hashują się różnie (`resource.c:1443` porównuje `view_type`). Komentarz w `vkd3d_private.h:1377-1379` nazywa to zamierzonym. To jest **odwrotność** niezmiennika upstreamu, i to ona umożliwia scenariusz z p. 2 (Emit postbuild kładący AS na buforze mikromapy).
- Nasze węzły histogramu żyją do końca życia widoku i są zwalniane w `vkd3d_view_destroy` (`resource.c:4975-4981`) — konstrukcja bez odpowiednika.

### Zapytania (postbuild)
- COMPACTED_SIZE: **zgodne** (typ zapytania, indeks puli, `VK_QUERY_RESULT_64_BIT | VK_QUERY_RESULT_WAIT_BIT`).
- SERIALIZATION: upstream **realne zapytanie** + zerowanie drugiego uint64; my **zaślepka** zerująca oba (`opacity_micromap.c:481-496`).
- CURRENT_SIZE i reszta: obie strony zerują, upstream `FIXME`, my `FIXME_ONCE`.
- `desc_offset` / stride 2×8: u nas nie istnieje, bo nie ma wariantu po tablicy VA.
- `d3d12_command_list_reset_query`: brak w obu naszych ścieżkach (AS i OMM) — **różnica wersji bazowej, nie odstępstwo**.

### Rejestracja rozszerzenia
Odwrócona (opt-out zamiast opt-in), flaga `dxr12` martwa, tier 1.2 promowany bezwarunkowo, dwa feature-structy zamiast jednego, plus sonda wybierająca backend. **To jest miejsce numer jeden do zmiany, jeśli nowa ścieżka ma być domyślnie wyłączona.**

### Przekazywanie do dxil-spirv
Aktywne u nas, `#if 0` u upstreamu, i **nasze jest bogatsze o dwa pola opcji**, których stub nie miał. Ścieżka wspólna dla obu backendów. Nie ruszać.

---

## 10. BLOKERY DLA WIERNEGO ODTWORZENIA (kolejność malejącej trudności)

1. **`vkd3d_view_map_create_view2` — czwarty argument rozjechał się nieodwracalnie.** upstream `bool rtas_is_omm` vs nasz `enum vkd3d_rtas_kind`. Nie da się mieć obu przez jedną flagę: albo trzeci wariant `create_view`, albo utrzymanie dwóch typów widoku i przyjęcie, że w tym punkcie odtworzenie NIE jest 1:1.
2. **`vkd3d_acceleration_structure_emit_postbuild_info` zmieniło kształt** — u nas `static`, pojedynczy VA, bez `count`/tablicy. Hunku #2515 nie da się nałożyć mechanicznie; rozgałęzienie AS/OMM trzeba wpiąć w nasz batchowany kształt.
3. **`d3d12_device_supports_ray_tracing_tier_1_2` trzeba dopisać** (nie ma go w drzewie) i przepiąć na nie 6 bramek, jeśli ścieżka ma odtwarzać upstreamową semantykę.
4. **`union vkd3d_omm_*`** — galęź `.ext` musi w środku wyglądać jak upstreamowe płaskie pole; unię można zachować.
5. **Petla naprawiająca `pNext` (:21475-21494) musi zostać**, nawet w wersji „upstreamowej", dopóki tablica linkage jest rezerwowana inkrementalnie. Upstream mógł jej nie mieć, bo rezerwował całość przed konwersją.

**Do wycięcia przy odtwarzaniu (kolejność testowania, od najgrubszego):**
1. Odpinanie wpięcia przy `OpacityMicromapArray == 0` (`opacity_micromap.c:641-646`) — zmienia geometrię widzianą przez sterownik.
2. Brak gałęzi OMM w `EmitRaytracingAccelerationStructurePostbuildInfo` — pozwala położyć AS na buforze mikromapy.
3. Drugi przebieg `resolve_omm_va_maps_ext` (`opacity_micromap.c:756`, wołany z `command.c:21906` i `device.c:9134`) + patch `0006`.
4. `return false` przy nieudanym placement (`opacity_micromap.c:706-711`).
5. Publikacja usageCounts na linkage BLAS (`opacity_micromap.c:684-686` + cała warstwa `:226-343`).
6. Odrzucanie `R8_UINT` (`opacity_micromap.c:615-628`).
7. Odroczenie postbuildu (`command.c:21739-21755` → `acceleration_structure.c:564-576`).

---

## 11. DODATKI DIAGNOSTYCZNE W DRZEWIE (pominięte w zestawieniu, do zdjęcia)

```
opacity_micromap.c:660-675   INFO("DIAG-ADDR: ...") — wypis wszystkich adresów urządzenia
opacity_micromap.c:688-694   komentarz POMIAR 11
opacity_micromap.c:699-705   komentarz POMIAR 12
opacity_micromap.c:782       INFO("POMIAR5-WPIECIE: va=...")
resource.c:5389-5398         INFO("POMIAR6-OK/NIEUDANE: vkCreateMicromapEXT ...")
va_map.c:472-480             komentarze POMIAR 3 i POMIAR 4
command.c:21673              INFO("POMIAR5-BUDOWA: va=...")
device.c:1457-1471           wymuszenie VKD3D_CONFIG_FLAG(FAULT) i (OMM_EXT_LINKAGE_USAGE_COUNTS)
```
Uwaga: `device.c:1470` **wymusza** `OMM_EXT_LINKAGE_USAGE_COUNTS`, czyli w obecnym buildzie publikacja usageCounts na linkage BLAS jest **aktywna**, mimo że kod projektowano jako domyślnie wyłączony.

Kopie sprzed sesji jako punkt odniesienia: `device.c.orig`, `vkd3d_private.h.orig`, `swapchain.c.orig` w `/var/home/michael/.git/proton-11/vkd3d-proton/libs/vkd3d/`.
Patche: `/var/home/michael/.git/proton-11/patches/vkd3d-proton/0005-vkd3d-ext-opacity-micromap-fallback.patch` (108106 B) i `0006-vkd3d-resolve-omm-handles-for-prebuild-sizing.patch` (6563 B).

Niczego nie modyfikowałem — wyłącznie Read, Grep i czytające Bash (`ls`, `wc`, `grep`, `sed -n`).
---

## STAN WYKONANIA LISTY Z SEKCJI 3 (2026-09-06)

| # | pozycja | stan |
|---|---|---|
| 1 | odpięcie struktury przy `OpacityMicromapArray == 0` | **cofnięte** — konwerter doczepia bezwarunkowo, jak upstream |
| 2 | gałąź OMM w `emit_postbuild_info` | **wstawione**; `write_postbuild_info_ext` nie jest już `static` |
| 3 | osobny przebieg rozwiązywania uchwytów | **usunięte** w całości wraz z deklaracją |
| 4 | `return false` przy nieudanym placement | **zamienione na `ERR` + przelot** |
| 5 | warstwa histogramu `usageCounts` | **usunięta** — funkcje, pole w widoku, pętla zwalniająca, flaga konfiguracyjna (73 flagi) |
| 6 | odrzucanie `DXGI_FORMAT_R8_UINT` | **przyjmowane** z `FIXME_ONCE`, jak upstream |
| 7 | odroczenie postbuildu po budowie OMM | **natychmiastowy** po `flush_rtas_batch`; gałąź `_ext` w partii zwinięta |
| 8 | zaślepka SERIALIZATION | **realne zapytanie** + zerowanie drugiego pola |
| 9 | `copy_ext` na widokach | **na uchwytach**; akcesory widokowe zeszły do `static` |
| 10 | kolejność dispatchy | **NIEZROBIONE ŚWIADOMIE** — patrz niżej |
| 11 | diagnostyka sesji | **zdjęta**, 0 wystąpień w binarce |
| 12 | pliki `.orig` | **usunięte** (16 szt. razem z `.rej`) |
| — | `ALLOW_OMM_LINKAGE_UPDATE` | **drugi bit + bramka**, z poprawką wobec planu |

### Pozycja 10 wstrzymana

Plan każe odwrócić kolejność tak, by `vkCmdBuildAccelerationStructuresKHR` szedł przed
`vkCmdBuildMicromapsEXT`. Komentarz w naszym `flush_rtas_batch` stwierdza, że partia nigdy nie
miesza obu typów, więc kolejność jest bez znaczenia — a gdyby kiedyś zmieszała, **nasza kolejność
jest poprawna (mikromapy przed strukturą, która je czyta), a upstreamowa nie.** Wierność nie jest
powodem, by przyjąć gorsze rozwiązanie. Do decyzji właściciela.

### Poprawka wobec treści planu — `ALLOW_OMM_LINKAGE_UPDATE`

Plan twierdzi, że „bity KHR i EXT są aliasami, więc pisownia `_KHR` jest równoważna". **Dla bitu
`DATA_UPDATE` to nieprawda.** W `khronos/Vulkan-Headers/include/vulkan/vulkan_core.h` istnieje
wyłącznie `VK_BUILD_ACCELERATION_STRUCTURE_ALLOW_OPACITY_MICROMAP_DATA_UPDATE_BIT_EXT` (0x100);
aliasu `_BIT_KHR` nie ma — promocja do KHR tego bitu nie przeniosła. Aliasem jest tylko
`..._OPACITY_MICROMAP_UPDATE_BIT` (0x40). Dlatego `DATA_UPDATE` jest bramkowany na
`d3d12_device_uses_ext_opacity_micromap`, a nie na samo `supports_opacity_micromap`.

Przy okazji ujawniło się, że oba bity mikromapowe były dotąd ustawiane **bez żadnej bramki**, czyli
także wtedy, gdy zgłaszamy brak obsługi OMM — utajone naruszenie VU na domyślnej ścieżce.
