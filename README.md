# ZMK-TRI-STATE

This module implements [Nick Conways's](https://github.com/nickconway) tri-state
behavior for [ZMK](https://github.com/zmkfirmware/zmk). This is a fork of
[Dhruvin Shah](https://github.com/dhruvinsh)'s original port. It adds a CI
pipeline that tests compatibility with new ZMK releases and creates matching
module releases.

## Usage

To load the module, add the following entries to `remotes` and `projects` in
`config/west.yml`.

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: urob
      url-base: https://github.com/urob
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: v0.3 # Set to desired ZMK release.
      import: app/west.yml
    - name: zmk-tri-state
      remote: urob
      revision: v0.3 # Should match ZMK release.
  self:
    path: config
```

## Summary

A tri-state is a key that triggers one behavior on the first press, another on
every press, and a third once a terminating condition is met.

`bindings` takes them in that order:

- **start** — fires once, on the first press.
- **continue** — fires on every press, including the first.
- **end** — fires once, when the tri-state ends (see below). Also called the
  'interrupt' behavior further down.

The canonical example is an alt-tab window switcher (see below for an improved
version):

```dts
/ {
    behaviors {
        swap: swapper {
            compatible = "zmk,behavior-tri-state";
            label = "SWAPPER";
            #binding-cells = <0>;
            bindings = <&kt LALT>, <&kp TAB>, <&kt LALT>;
        };
    };
```

Pressing `&swap` the first time toggles `LALT` down and then taps `TAB`, 
every subsequent `&swap` press taps `TAB` again, and terminating 
the tri-state behavior toggles `LALT` off.

Specifically, a tri-state ends if any of the following occurs:

- An event at a key position not listed in `ignored-key-positions`. Releases
  count as well as presses, so releasing a held layer key ends the tri-state.
  The tri-state's own key position never ends it.
- The activation of a layer not listed in `ignored-layers`, or the deactivation
  of a layer listed in `end-on-layer-deactivation`. All other layer
  deactivations are ignored.
- `timeout-ms` elapsing after the tri-state key, or an ignored key, is released.

## Configuration

### `timeout-ms`

Setting `timeout-ms` will cause the end behavior to fire once the time has
elapsed after releasing the tri-state or an ignored key.

### `ignored-key-positions`

- Including `ignored-key-positions` in your tri-state definition will let the
  key positions specified NOT trigger the interrupt behavior when a tri-state is
  active.
- Pressing any key **NOT** listed in `ignored-key-positions` will cause the
  interrupt behavior to fire.
- Note that `ignored-key-positions` is an array of key position indexes. Key
  positions are numbered according to your keymap, starting with 0. So if the
  first key in your keymap is Q, this key is in position 0. The next key
  (probably W) will be in position 1, et cetera.
- Extending the swapper above with a backwards-tab key gives the popular
  [Swapper](https://github.com/callum-oakley/qmk_firmware/tree/master/users/callum)
  from Callum Oakley:

```dts
/ {
    behaviors {
        swap: swapper {
            compatible = "zmk,behavior-tri-state";
            label = "SWAPPER";
            #binding-cells = <0>;
            bindings = <&kt LALT>, <&kp TAB>, <&kt LALT>;
            ignored-key-positions = <1>;
        };
    };
    keymap {
        compatible = "zmk,keymap";
        label ="Default keymap";
        default_layer {
            bindings = <
                &swap    &kp LS(TAB)
                &kp B    &kp C>;
        };
    };
};
```

- The sequence `(swap, swap, LS(TAB))` produces
  `(LA(TAB), LA(TAB), LA(LS(TAB)))`. The LS(TAB) behavior does not fire the
  interrupt behavior, because it is included in `ignored-key-positions`.
- The sequence `(swap, swap, B)` produces `(LA(TAB), LA(TAB), B)`. The B
  behavior **does** fire the interrupt behavior, because it is **not** included
  in `ignored-key-positions`.
- In practice, the swapper is typically placed on a momentary layer and is 
  terminated by releasing that layer.

### `ignored-layers`

- By default, any layer *activation* will trigger the end behavior. Layer
  deactivations are ignored unless listed in `end-on-layer-deactivation`.
- Including `ignored-layers` in your tri-state definition will let the specified
  layers NOT trigger the end behavior when they become active.
- List every layer that can be activated while the tri-state is running. That
  includes layers opened by the tri-state's own start behavior (e.g. a `&tog`),
  and layers opened by a key in `ignored-key-positions`. 
  A layer only needs listing if it can be activated
  *during* the tri-state; one that was already active when the tri-state
  started does not.
- Activating any layer **NOT** listed in `ignored-layers` will cause the
  interrupt behavior to fire.
- Note that `ignored-layers` is an array of layer indexes. Layers are numbered
  according to your keymap, starting with 0. The first layer in your keymap is
  layer 0. The next layer will be layer 1, et cetera.
- The most common use is a tri-state that opens a layer itself and ends the layer
  when any key not whitelisted by `ignored-key-positions` is pressed. Here
  `ignored-layers` is not optional: without it the tri-state would see its own
  start behavior activate layer 1 and immediately fire the end behavior.

```dts
/ {
    behaviors {
        smart_layer: smart_layer {
            compatible = "zmk,behavior-tri-state";
            label = "SMART_LAYER";
            #binding-cells = <0>;
            bindings = <&tog 1>, <&none>, <&tog 1>;
            ignored-key-positions = <1 3>;
            ignored-layers = <1>;
        };
    };
    keymap {
        compatible = "zmk,keymap";
        label ="Default keymap";
        default_layer {
            bindings = <
                &smart_layer  &kp A
                &kp B         &kp C>;
        };
        second_layer {
            bindings = <
                &trans        &kp LEFT
                &kp B         &kp RIGHT>;
        };
    };
};
```

- Pressing `smart_layer` toggles layer 1 on and keeps it there. `LEFT` and
  `RIGHT` are in `ignored-key-positions`, so they can be used freely; `B` is
  not, so pressing it fires the end behavior and toggles layer 1 back off.

### `end-on-layer-deactivation`

- By default, layer deactivations never end the tri-state, only activations do
  (see `ignored-layers`).
- Layers listed here end the tri-state when they are deactivated. Useful when a
  layer can drop with no key event left to interrupt, for example num-word
  dropping the layer on the tri-state's own keycode, which otherwise leaves the
  tri-state held until `timeout-ms`.
- Like `ignored-layers`, this is an array of layer indexes. The two lists are
  independent of each other.

## References

- Nick Conway's original behavior
  [PR](https://github.com/zmkfirmware/zmk/pull/1366).
- Dhruvin Shah's [module port](https://github.com/dhruvinsh/zmk-tri-state).
- [Pipeline](https://github.com/urob/zmk-actions) used for automated testing and
  releases.
- My personal [zmk-config](https://github.com/urob/zmk-config) contains advanced
  usage examples.
