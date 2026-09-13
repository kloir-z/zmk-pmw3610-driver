## このフォークについて

`scroll-acceleration-feature` ブランチに、トラボスクロールの改善を入れています（本家への PR: [kumamuk-git/zmk-pmw3610-driver#4](https://github.com/kumamuk-git/zmk-pmw3610-driver/pull/4)）。

- 素早く動かしたときにスクロールが途切れないよう、しきい値を超えた分を次に繰り越します
- 動かす速さに応じてスクロール量を増やす加速機能を追加しています（任意）

使う場合は `config/west.yml` の remotes に kloir-z を足し、ドライバの参照先を書き換えてください。

```yml
  remotes:
    - name: kloir-z
      url-base: https://github.com/kloir-z
  projects:
    - name: zmk-pmw3610-driver
      remote: kloir-z
      revision: scroll-acceleration-feature
```

加速を使う場合は `roBa_R.conf` に追加します。書かなくても繰り越しの改善は効きます。

```conf
CONFIG_PMW3610_SCROLL_ACCELERATION=y
# 1〜10、大きいほど強く加速（既定は3）
CONFIG_PMW3610_SCROLL_ACCELERATION_SENSITIVITY=3
```

---

PMW3610 driver implementation for ZMK with at least Zephyr 3.5

This work is based on [ufan's implementation](https://github.com/ufan/zmk/tree/support-trackpad) of the driver.

## Installation

Only GitHub actions builds are covered here. Local builds are different for each user, therefore it's not possible to cover all cases.

Include this project on your ZMK's west manifest in `config/west.yml`:

```yml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/petejohanson
    - name: inorichi
      url-base: https://github.com/inorichi
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: feat/pointers-move-scroll
      import: app/west.yml
    - name: zmk-pmw3610-driver
      remote: inorichi
      revision: main
  self:
    path: config
```

Then, edit your `build.yml` to look like this, 3.5 is now on main:

```yml
on: [workflow_dispatch]

jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@main
```

Now, update your `board.overlay` adding the necessary bits (update the pins for your board accordingly):

```dts
&pinctrl {
    spi0_default: spi0_default {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 8)>,
                <NRF_PSEL(SPIM_MOSI, 0, 17)>,
                <NRF_PSEL(SPIM_MISO, 0, 17)>;
        };
    };

    spi0_sleep: spi0_sleep {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 8)>,
                <NRF_PSEL(SPIM_MOSI, 0, 17)>,
                <NRF_PSEL(SPIM_MISO, 0, 17)>;
            low-power-enable;
        };
    };
};

&spi0 {
    status = "okay";
    compatible = "nordic,nrf-spim";
    pinctrl-0 = <&spi0_default>;
    pinctrl-1 = <&spi0_sleep>;
    pinctrl-names = "default", "sleep";
    cs-gpios = <&gpio0 20 GPIO_ACTIVE_LOW>;

    trackball: trackball@0 {
        status = "okay";
        compatible = "pixart,pmw3610";
        reg = <0>;
        spi-max-frequency = <2000000>;
        irq-gpios = <&gpio0 6 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;

        /*   optional features   */
        // snipe-layers = <1>;
        // scroll-layers = <2 3>;
        // automouse-layer = <4>;

        /*   optional: ball action on specific layers  */
        // arrows {
        //     layers = <3>;
        //     bindings =
        //         <&kp RIGHT_ARROW>,
        //         <&kp LEFT_ARROW>,
        //         <&kp UP_ARROW>,
        //         <&kp DOWN_ARROW>;
        //
        //     /*   optional: ball action configuration  */
        //     tick = <10>;
        //     // wait-ms = <5>;
        //     // tap-ms = <5>;
        // };
    };
};

/ {
  trackball_listener {
    compatible = "zmk,input-listener";
    device = <&trackball>;

  };
};
```

Now enable the driver config in your `board.config` file (read the Kconfig file to find out all possible options):

```conf
CONFIG_SPI=y
CONFIG_INPUT=y
CONFIG_ZMK_MOUSE=y
CONFIG_PMW3610=y
```
