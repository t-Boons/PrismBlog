---
layout: default
title: Prism - Documentation
---
# Prism Lens-Flare renderer.
Prism is a lens flare editor plugin that allows the user to create customized lens-flares inside of Unreal Engine 5. The plugin improves the existing Lens-Flare rendering post-process and allows for more visual and creative freedom. In this blog I will go over some of the systems and things I have learned throughout the creation of this plugin.

*Screenshot of the plugin in action*
![](assets/Showcase1.gif)

## Instanced rendering
The lens-flares are rendered per instance, reason being that the only shape a lens-flare bokeh can have is a rectangle. Since we only have to render quads at different locations we don't have to mess around with vertex buffers and can hard code the vertices for a quad inside of the vertex shader itself, after which we can do N number of draw-calls and use the instance index to transform the vertices into the right position.

The only thing the GPU has to know about is the per-bokeh struct that contains information about said lens-flare bokeh.

*This is a highly stripped down version of the rendering code inside PrismRenderer.cpp.*
```cpp
    // Create the SRV used to pass the bokeh data to the shader.
    FRDGBufferSRVRef Srv = GraphBuilder.CreateSRV(CreateStructuredBuffer(GraphBuilder, TEXT("PrismRendererBokehs"), BokehGPUData));

    // Set up vertex shader parameters.
    FPrismRendererVS::FParameters VSParameters;
    VSParameters.BokehDataBuffer = Srv;
    VSParameters.NumBokehData = BokehGPUData.Num();

    FPrismRendererPS::FParameters PSParameters;
    PSParameters.RenderTargets[0] = FRenderTargetBinding(SceneColor.Texture, ERenderTargetLoadAction::ELoad);
    PSParameters.BokehDataBuffer = Srv;
    PSParameters.NumBokehData = BokehGPUData.Num();
    // Conbine pixel and vertex shader parameters.
    FPRismRendererShaderParameters* Parameters = GraphBuilder.AllocParameters<FPRismRendererShaderParameters>();
    Parameters->VS = VSParameters;
    Parameters->PS = PSParameters;

    // Add the lens flare pass to the render graph.
    GraphBuilder.AddPass(
        RDG_EVENT_NAME("PrismLensFlarePass"),
        Parameters,
        ERDGPassFlags::Raster,
        [this, PSParameters, Parameters, SceneColor](FRHICommandListImmediate& RHICmdList)
        {
            // Set the shader parameters.
            SetShaderParameters(RHICmdList, VertexShader, VertexShader.GetVertexShader(), Parameters->VS);
            SetShaderParameters(RHICmdList, PixelShader, PixelShader.GetPixelShader(), Parameters->PS);

            // Submit instanced draw call based on how many bokehs there are.
            RHICmdList.SetStreamSource(0, nullptr, 0);
            RHICmdList.DrawPrimitive(0, 2, Parameters->VS.NumBokehData);
        });
```

## FSceneViewExtentionBase
FSceneViewExtentionBase is the class used by Unreal Engine to hook into the post-processing pipeline. In this plugin it is used to hook in the pipeline right before the bloom pass. This is where Unreal Engine itself renders its built in lens-flares hence we do the same.

By overriding the `SubscribeToPostProcessingPass` function we can bind a delegate that will be ran before the bloom pass.

*SubscribeToPostProcessingPass function override in PrismViewExtention.cpp*
```cpp
void FPrismViewExtention::SubscribeToPostProcessingPass(EPostProcessingPass Pass, const FSceneView& InView, FPostProcessingPassDelegateArray& InOutPassCallbacks, bool bIsPassEnabled)
{
    // MotionBlur pass is marked as BL_SceneColorBeforeBloom. So it runs before the Bloom pass.
    // The bloom pass contains the existing lens-flare pass hence we try to get the delegates for the MotionBlur pass.
    if (Pass == EPostProcessingPass::MotionBlur)
    {
        InOutPassCallbacks.Add(FPostProcessingPassDelegate::CreateRaw(this, &FPrismViewExtention::RenderBeforeBloom_RenderThread));
    }
}
```

The `RenderBeforeBloom_RenderThread` function will be ran every time before the bloom pass. And will get the sun light data from the active frame by getting it from the `FPrismLightCollector`.

## Light line-trace querying
We want to get the light's position relative to the camera every frame. We also want to know if anything is obscuring the camera. For this we use a line-trace however, this was more difficult than expected, because we now have to 1: query the line-trace on the game thread and 2: make sure that the render function is querying from the right frame otherwise we get flickering and other graphical bugs.

I did this by first calling a line-trace query on the game thread for that frame, saving that query in a `TMap<>` where we use the `FSceneView::GetViewKey()` as the key for the query. After which the render thread can fetch this whenever it wants.

*SetupView function inside of PrismViewExtention.cpp*
```cpp
void FPrismViewExtention::SetupView(FSceneViewFamily& InViewFamily, FSceneView& InView)
{
	UPrismRendererSubSystem* SubSystem = GEngine->GetEngineSubsystem<UPrismRendererSubSystem>();
    if (SubSystem)
    {
        LightCollector.EnqueLightDataRequest_GameThread(InView);
    }
}
```

*RenderBeforeBloom_RenderThread function inside of PrismViewExtention.cpp*
```cpp
FScreenPassTexture FPrismViewExtention::RenderBeforeBloom_RenderThread(FRDGBuilder& GraphBuilder, const FSceneView& View, const FPostProcessMaterialInputs& InOutInputs)
{
	check(IsInRenderingThread());

	FScreenPassTexture SceneColor = InOutInputs.ReturnUntouchedSceneColorForPostProcessing(GraphBuilder);
	const TArray<FPrismScreenLight> Lights = LightCollector.GetLightDataRequest_RenderThread(View);
	Renderer.Render_RenderThread(GraphBuilder, SceneColor, Lights);

	return SceneColor;
}
```

## Custom asset management.
Prism uses a custom `UPrismLensFlareAsset` asset that contains all the data necessary to render the lens-flare bokehs.
To allow for this we use several classes, including a: FAssetTypeActions class, PrismAssetFactory class and the actual UPrismLensFlareAsset class.

*Custom asset creation classes.*
```cpp
/// <summary>
/// FAssetTypeActions_PrismAsset is used to register the PrismLensFlareAsset type and make it available in the editor when right clicking to create a new asset.
/// </summary>
class PRISMEDITOR_API FAssetTypeActions_PrismAsset : public FAssetTypeActions_Base
{
public:
    virtual FText GetName() const override { return NSLOCTEXT("Prism LensFlare Asset", "AssetTypeActions_Prism", "Prism LensFlare Asset"); }
    virtual FColor GetTypeColor() const override { return FColor(255, 200, 50); }
    virtual UClass* GetSupportedClass() const override { return UPrismLensFlareAsset::StaticClass(); }
    virtual uint32 GetCategories() override { return EAssetTypeCategories::Misc; }
    virtual void GetActions(const TArray<UObject*>& InObjects, FMenuBuilder& MenuBuilder) override;
};
```

```cpp
/// <summary>
/// PrismAssetfactory is required to create a new PrismLensFlareAsset in the editor.
/// </summary>
UCLASS()
class PRISMEDITOR_API UPrismAssetFactory : public UFactory
{
	GENERATED_BODY()

public:
	UPrismAssetFactory();

	virtual UObject* FactoryCreateNew(UClass* Class, UObject* Parent, FName Name, EObjectFlags Flags, UObject* Object, FFeedbackContext* FeedbackContext) override;
};
```

```cpp
DECLARE_MULTICAST_DELEGATE(FOnAssetChanged);
DECLARE_MULTICAST_DELEGATE(FOnDeleting);
DECLARE_MULTICAST_DELEGATE(FOnAssetRenamed);
/// <summary>
/// UPrismLensFlareAsset is used to store an array of lens flare bokeh data as an asset.
/// </summary>
UCLASS(BlueprintType)
class PRISM_API UPrismLensFlareAsset : public UObject
{
	GENERATED_BODY()

public:

#if WITH_EDITOR
	/// <summary>
	/// Broadcasts the OnAssetChanged event.
	/// </summary>
	virtual void PostEditChangeProperty(FPropertyChangedEvent& PropertyChangedEvent) override;

	/// <summary>
	/// Broadcasts the OnAssetBeingDeleted event.
	/// </summary>
	virtual void BeginDestroy() override;

	/// <summary>
	/// Called when asset is renamed.
	/// </summary>
	virtual void PostRename(UObject* OldOuter, const FName OldName) override;
#endif

public:
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "PrismAsset")
	TArray<FPrismLensFlareAssetData> Bokehs;

	FOnAssetRenamed OnAssetRenamed;
	FOnAssetChanged OnAssetChanged;
	FOnDeleting OnAssetBeingDeleted;
};
```

Along with the custom asset there is also a PrismEditor module that has the code to allow the user to set the global lens-flare used for rendering during editing. This will also save the asset to a custom config file.

*Menu entry code for custom lens-flare asset inside of AssetTypeActions_PrismAsset.cpp*
```cpp
void FAssetTypeActions_PrismAsset::GetActions(const TArray<UObject*>& InObjects, FMenuBuilder& MenuBuilder)
{
	TArray<TWeakObjectPtr<UPrismLensFlareAsset>> Assets = GetTypedWeakObjectPtrs<UPrismLensFlareAsset>(InObjects);

	// Early out if the asset array is empty.
	if(Assets.IsEmpty())
	{
		return;
	}

	// Get the frist element.
	TWeakObjectPtr<UPrismLensFlareAsset> FirstAsset = Assets[0];

	// Add Menu Entry button when right-clicking the asset.
	MenuBuilder.AddMenuEntry(
		NSLOCTEXT("AssetTypeActions", "Prism_DoSomething", "Use this Asset"),
		NSLOCTEXT("AssetTypeActions", "Prism_DoSomethingTooltip", "Sets the currently selected Prism asset to the one used."),
		FSlateIcon(FAppStyle::GetAppStyleSetName(), "Icons.Play"),
		FUIAction(
			FExecuteAction::CreateLambda([FirstAsset]()
				{
					UPrismAssetSubSystem* SubSystem = GEngine->GetEngineSubsystem<UPrismAssetSubSystem>();
					if (SubSystem)
					{
						SubSystem->SetGlobalPrismLensFlareAssetAndSaveToConfig(FirstAsset.Get());
					}
				})));
}
```

