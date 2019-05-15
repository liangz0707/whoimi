# GPU架构整合

AMD公司GPU芯片架构:

| Microarchitecture |                             GPUs                             |                     Graphic cards / SoCs                     |
| :---------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|      GCN 5th      |                           Vega 10                            |                    Radeon Rx Vega series                     |
|      GCN 4th      |              Polaris 10, Polaris 11, Polaris 12              |            Radeon Rx 400 series Radeon_500_series            |
|      GCN 3rd      |                     Tonga, Fiji, Carrizo                     |                       Radeon R9 Series                       |
|      GCN 2nd      | Bonaire, Hawaii, Kaveri, Kabini, Temash, Mullins, Beema, Carrizo-L |         Radeon HD 7790, **PlayStation 4**, Xbox One          |
|      GCN 1st      |             Oland, Cape Verde, Pitcairn, Tahiti              |                  Radeon HD 77xx–7900 Series                  |
|    TeraScale 3    |                   Cayman, Trinity/Richland                   |      Radeon HD 69xx Series, Radeon HD 7xxx–76xx Series       |
|    TeraScale 2    |         Cedar, Cypress, Juniper, Redwood, Palm, Sumo         | Radeon HD 5000 Series, Radeon HD 6350, Radeon HD 64xx–68xx Series |
|    TeraScale 1    |             R600, RV630, RV610, RV790, RV770, …              |           Radeon HD 2000 Series, HD 3000, HD 4000            |

Nvidia公司GPU芯片架构:

| Microarchitecture | GPUs                   | Graphic cards / SoCs                                         |
| ----------------- | ---------------------- | ------------------------------------------------------------ |
| Turing            | TU10x, TU11x           | GeForce 20 series, GeForce 16 series                         |
| Volta             | GV10x                  | Nvidia Titan V                                               |
| Pascal            | GP10x                  | GeForce 10 series, Tegra X2                                  |
| Maxwell           | GM10x, GM20x           | GeForce GTX 750 Ti, GTX 750, GTX 860M, GeForce 900 series, Tegra X1 |
| Kepler            | GK10x, GK110, GK208    | GeForce 600 series, GeForce 700 series, Tegra K1             |
| Fermi             | GF10x, GF11x           | GeForce 400 series, GeForce 500 series                       |
| Tesla             | G8x, G9x, GT20x, GT21x | GeForce 8 series, GeForce 9 series, GeForce 100 series, GeForce 200 series, GeForce 300 series |

高通公司GPU架构:

| Microarchitecture | GPUs                                         | Graphic cards / SoCs                                        |
| ----------------- | -------------------------------------------- | ----------------------------------------------------------- |
| Adreno 500 series | Adreno 510, Adreno 530, Adreno 540Adreno 512 | Snapdragon 430, 625, 650, 652, 660,820, 821, 835            |
| Adreno 400 series | Adreno 418,Adreno 420,Adreno 430             | Snapdragon 415, 615, 616, 617, 805, 808, 810                |
| Adreno 300 series |                                              | Snapdragon 200, 208, 210, 212, 400, 410, 412, 600, 800, 801 |

ARM公司GPU架构:

| ARM         | GPUs                            | Graphic cards / SoCs                                         |
| ----------- | ------------------------------- | ------------------------------------------------------------ |
| Bifrost     | Mali-G71, …                     | Kirin 960, 970, Exynos 8895, MediaTek Helio P23 (MT6763T), Helio P30 |
| Midgard 4th | Mali-T860, Mali-T830, Mali-T880 | Exynos 8890, Exynos 7880, Exynos 7870, Kirin 950, 955, MediaTek MT6738, MT6750, Helio X20 (MT6797), X25 (MT6797T), P10 (MT6755), P20 (MT6757) |
| Midgard 3rd | Mali-T760, …                    | Exynos 7420, Exynos 5433, MT6752, MT6732, RK3288             |
| Midgard 2nd | Mali-T600 series, T720          | Exynos 5250, 5260, 5410, 5420, 5422, 5430, 5800, 7580, Mediatek MT6735, MT6753, Kirin 920, 925, 930, 935 |