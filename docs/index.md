# Milk-V Mars GPU bring-up

This project documents bringing up the PowerVR BXE-4-32 GPU and DC8200/HDMI display on the Milk-V Mars (JH7110). The work is prepared on ubhejane before testing on the board.

Start with the [project plan](plan.md), then follow the [host setup](host-setup.md) and [pinned sources](sources.md). The completed work is documented in the [devicetree analysis](dts-notes.md), [kernel build](kernel.md), [QEMU smoke test](qemu-smoke.md), and [boot firmware build](boot.md).

To view this documentation locally, install the dependencies with `python3 -m pip install -r requirements-docs.txt`, then run `mkdocs serve` from the project root. Run `mkdocs build --strict` to check the site.
