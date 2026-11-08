# Dependency audit: no ImageSharp

CrystalRenderer references KamiToolKit **2.2.66**, which has no package dependencies.
KamiToolKit 1.1.2 declared SixLabors.ImageSharp 3.1.12, whose NuGet advisories
(GHSA-gwg2-r3hj-4w44, GHSA-j3p4-wp97-rph4, GHSA-j9gm-c75j-xc9q, GHSA-jjfr-hcj7-qf5w,
GHSA-v76p-62qx-wwq2, GHSA-wmxv-xphr-5c9g) failed audited restores under
warnings-as-errors. CrystalRenderer never called ImageSharp itself.

The 2.x port follows renamed namespaces (`KamiToolKit.UiOverlay`,
`KamiToolKit.Nodes.Simplified`), the single-argument node `Dispose`, and the
asynchronous `KamiToolKitLibrary.InitializeAsync`. `DalamudScreenRenderer` starts
KamiToolKit in the background; until the native overlay exists, or if it cannot be
created, `IsNativeOverlayAvailable` is false and frames fall back to the ImGui overlay.

No advisories are suppressed and auditing is unchanged. The native overlay output still
needs an in-game check after this upgrade.
