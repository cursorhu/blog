# Chat Transcript - Tab 1

## User:


[text]  
@Codebase @Web ����һ��Storport miniport �������һ��ɵ�Դ������ع��������⣬ ���ѯStorport�ĵ��Լ�Windows driver Pofx��DFx(Direct Pofx)�ĵ�����������


## AI (claude-3.5-sonnet):

��������һ����������ĵ�Դ������ش���:

1. ��Ҫ�ĵ�Դ�����������¼����ļ���:


```35:404:SDSTORDriver/sdstor/main/pmfunc.c
void pm_init(bht_dev_ext_t * pdx)
{

    DbgInfo(MODULE_MAIN_PM, FEATURE_DRIVER_INIT, 0, "Enter %s chip:%xh\n", __FUNCTION__, pdx->host.chip_type);

    //TODO: RTD3 enable From the registry
    pdx->pm_state.s3s4_entered = FALSE;
    pdx->pm_state.rtd3_en = hostven_rtd3_check(&pdx->host);
    pdx->pm_state.rtd3_entered = FALSE;
    pdx->pm_state.s5_entered = FALSE;
    pdx->pm_state.warm_boot_entered = FALSE;
    hostven_pm_mode_cfg(&pdx->host, &(pdx->pm_state));
    os_pm_init(pdx);

    DbgInfo(MODULE_MAIN_PM, FEATURE_PM_TRACE, 0, "Exit %s\n", __FUNCTION__);
}

 void SetSlowLTRNoRequire(sd_host_t * host)
{
	if (host->cfg->issue_fix_item.ltr_c10_patch.enable_ltr_patch)
	{

		if (host->chip_type == CHIP_FUJIN2 || host->chip_type == CHIP_SEABIRD)
		{
            pci_andl(host, 0x3e8, 0x7fff7fff);
		
            pci_orl(host, 0x3e8, 0x20000000);
             DbgInfo(MODULE_SD_HOST, FEATURE_CARD_INIT , NOT_TO_RAM,"SetSlowLTRNoRequire\n");
			//DbgInfo(("no LTR\n"));
		}
	}
	
}

 void SetSlowLTRRequire_when_Enabledriver(sd_host_t* host)
 {
     if (!host->cfg->issue_fix_item.ltr_c10_patch.enable_ltr_patch)
     {

         if (host->chip_type == CHIP_FUJIN2 || host->chip_type == CHIP_SEABIRD)
         {
             pci_orw(host, 0x3e8, 0x8000);
             pci_orl(host, 0x3e8, 0x20000000);
             DbgInfo(MODULE_SD_HOST, FEATURE_CARD_INIT, NOT_TO_RAM, "SetSlowLTRNoRequire When Enable Driver\n");
         }
     }

 }
...
void pcr_part_a_backup(bht_dev_ext_t * pdx)
{
    if(pdx->cfg->driver_item.backup_part_a)
    {
        //enable or disable LED output or not.
        pdx->pm_state.reg_0xdc = pci_readl(&pdx->host, 0xdc);

        //disable or enable RTD3
        pdx->pm_state.reg_0x3e0	= pci_readl(&pdx->host, 0x3e0);

        pdx->pm_state.reg_0x3e8 = pci_readl(&pdx->host, 0x3e8);

        pdx->pm_state.reg_0x3ec = pci_readl(&pdx->host, 0x3ec);

        //Change the ASPM L0s Exit Latency
        pdx->pm_state.reg_0xf4 = pci_readl(&pdx->host, 0xf4);

        //ep nfts value
        pdx->pm_state.reg_0x74  = pci_readl(&pdx->host, 0x74);

        //Power saving mode setting
        pdx->pm_state.reg_0xf0 = pci_readl(&pdx->host, 0xf0);

        //enable or disable ASPM L0s&L1
        pdx->pm_state.reg_0x90  =  pci_readl(&pdx->host, 0x90);

    }

}

void pcr_part_b_backup(bht_dev_ext_t * pdx)
{
    if(pdx->cfg->driver_item.backup_part_b)
    {
        pdx->pm_state.reg_0x64 = pci_readl(&pdx->host, 0x64);
        //skt power control output enable
        pdx->pm_state.reg_0xec = pci_readl(&pdx->host, 0xec);
        //ocb cntl timer
        //ocb cntl enable
        pdx->pm_state.reg_0xd4 = pci_readl(&pdx->host, 0xd4);
        pdx->pm_state.reg_0x3e4 = pci_readl(&pdx->host, 0x3e4);
        //PLL DM
        pdx->pm_state.reg_0x304 = pci_readl(&pdx->host, 0x304);

        //Set Base Clock Frequency
        pdx->pm_state.reg_0x328 = pci_readl(&pdx->host, 0x328);

        //Set DLL tuning window
        pdx->pm_state.reg_0x300 = pci_readl(&pdx->host, 0x300);

        //aux power LDO
        pdx->pm_state.reg_0x68 = pci_readl(&pdx->host, 0x68);

        //Adjust the output delay for SD2.0 high speed mode
        pdx->pm_state.reg_0x350 = pci_readl(&pdx->host, 0x350);
        //T_EIDL_ENTRY
        pdx->pm_state.reg_0x35c = pci_readl(&pdx->host, 0x35c);

        //disable bit for UHSII term_resistor_calibration
        pdx->pm_state.reg_0x3e0 = pci_readl(&pdx->host, 0x3e0);

        //Set the Max power supply capability of SD host
        pdx->pm_state.reg_0x334	= pci_readl(&pdx->host, 0x334);

        //external enable polarity control pin
        pdx->pm_state.reg_0xd8 = pci_readl(&pdx->host, 0xd8);

        //AOSC off support
        pdx->pm_state.reg_0x3f0 = pci_readl(&pdx->host, 0x3f0);

        //max read request size
        pdx->pm_state.reg_0x88 =  pci_readl(&pdx->host, 0x88);

        //UHSII DLL watch dog
        pdx->pm_state.reg_0x33c = pci_readl(&pdx->host, 0x33c);

        //PCI-PM L1 entrance timer
        pdx->pm_state.reg_0xe0 = pci_readl(&pdx->host, 0xe0);

        //ASPM L1 entrance timer
        pdx->pm_state.reg_0xfc = pci_readl(&pdx->host, 0xfc);

    }
}

void pcr_part_a_restore(bht_dev_ext_t * pdx)
{
    if(pdx->cfg->driver_item.backup_part_a)
    {

        /*
        The register mapping way to write register must be in D0 state(at which device register is mapped to PCIe BAR space).
        In D3 state, the register access is invalid and return 0xFF.
        Refer to:
        StorPortWriteRegisterBufferUlong: The given Register must be in a mapped memory space range that was returned by StorPortGetDeviceBase
     
        Usually this ScsiSetRunningConfig is after receive ScsiAdapterPower - D0 and it's in D0 state, 
        but in some platform AMD or BIOS difference, the operation is in D3 (storport allows it)
        When PCI config register is accessed in D3, must use PORT access to write register, StorPortSetBusDataByOffset can only be used in ScsiSetRunningConfig in AdapterControl.
        Refer to:
        https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ne-storport-scsi_adapter_control_type
        https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportsetbusdatabyoffset
        */
        if (pdx->pm_state.d0d3_state == StorPowerDeviceD0) {
            pci_writel(&pdx->host, 0xdc, pdx->pm_state.reg_0xdc);
            pci_writel(&pdx->host, 0x3e0, pdx->pm_state.reg_0x3e0);
            pci_writel(&pdx->host, 0x3e8, pdx->pm_state.reg_0x3e8);
            pci_writel(&pdx->host, 0x3ec, pdx->pm_state.reg_0x3ec);
            pci_writel(&pdx->host, 0xf4, pdx->pm_state.reg_0xf4);
            pci_writel(&pdx->host, 0x74, pdx->pm_state.reg_0x74);
            pci_writel(&pdx->host, 0xf0, pdx->pm_state.reg_0xf0);
            pci_writel(&pdx->host, 0x90, pdx->pm_state.reg_0x90);
        }
        else { //D3
            pci_cfgio_writel(&pdx->host, 0xdc, pdx->pm_state.reg_0xdc);
            pci_cfgio_writel(&pdx->host, 0x3e0, pdx->pm_state.reg_0x3e0);
            pci_cfgio_writel(&pdx->host, 0x3e8, pdx->pm_state.reg_0x3e8);
            pci_cfgio_writel(&pdx->host, 0x3ec, pdx->pm_state.reg_0x3ec);
            pci_cfgio_writel(&pdx->host, 0xf4, pdx->pm_state.reg_0xf4);
            pci_cfgio_writel(&pdx->host, 0x74, pdx->pm_state.reg_0x74);
            pci_cfgio_writel(&pdx->host, 0xf0, pdx->pm_state.reg_0xf0);
            pci_cfgio_writel(&pdx->host, 0x90, pdx->pm_state.reg_0x90);
        }
        
    }
}


void pcr_part_b_restore(bht_dev_ext_t * pdx)
{
    if(pdx->cfg->driver_item.backup_part_b)
    {

        if (pdx->pm_state.d0d3_state == StorPowerDeviceD0) {
            pci_writel(&pdx->host, 0x64, pdx->pm_state.reg_0x64);
            pci_writel(&pdx->host, 0xec, pdx->pm_state.reg_0xec);
            pci_writel(&pdx->host, 0xd4, pdx->pm_state.reg_0xd4);
            pci_writel(&pdx->host, 0x304, pdx->pm_state.reg_0x304);
            pci_writel(&pdx->host, 0x328, pdx->pm_state.reg_0x328);
            pci_writel(&pdx->host, 0x300, pdx->pm_state.reg_0x300);
            pci_writel(&pdx->host, 0x3e4, pdx->pm_state.reg_0x3e4);
            pci_writel(&pdx->host, 0x68, pdx->pm_state.reg_0x68);

            pci_writel(&pdx->host, 0x350, pdx->pm_state.reg_0x350);
            pci_writel(&pdx->host, 0x35c, pdx->pm_state.reg_0x35c);
            pci_writel(&pdx->host, 0x3e0, pdx->pm_state.reg_0x3e0);
            pci_writel(&pdx->host, 0x334, pdx->pm_state.reg_0x334);
            pci_writel(&pdx->host, 0xd8, pdx->pm_state.reg_0xd8);
            pci_writel(&pdx->host, 0x3f0, pdx->pm_state.reg_0x3f0);
            pci_writel(&pdx->host, 0x88, pdx->pm_state.reg_0x88);
            pci_writel(&pdx->host, 0x33c, pdx->pm_state.reg_0x33c);
            pci_writel(&pdx->host, 0xe0, pdx->pm_state.reg_0xe0);
            pci_writel(&pdx->host, 0xfc, pdx->pm_state.reg_0xfc);
        }
        else {
            pci_cfgio_writel(&pdx->host, 0x64, pdx->pm_state.reg_0x64);
            pci_cfgio_writel(&pdx->host, 0xec, pdx->pm_state.reg_0xec);
            pci_cfgio_writel(&pdx->host, 0xd4, pdx->pm_state.reg_0xd4);
            pci_cfgio_writel(&pdx->host, 0x304, pdx->pm_state.reg_0x304);
            pci_cfgio_writel(&pdx->host, 0x328, pdx->pm_state.reg_0x328);
            pci_cfgio_writel(&pdx->host, 0x300, pdx->pm_state.reg_0x300);
            pci_cfgio_writel(&pdx->host, 0x3e4, pdx->pm_state.reg_0x3e4);
            pci_cfgio_writel(&pdx->host, 0x68, pdx->pm_state.reg_0x68);

            pci_cfgio_writel(&pdx->host, 0x350, pdx->pm_state.reg_0x350);
            pci_cfgio_writel(&pdx->host, 0x35c, pdx->pm_state.reg_0x35c);
            pci_cfgio_writel(&pdx->host, 0x3e0, pdx->pm_state.reg_0x3e0);
            pci_cfgio_writel(&pdx->host, 0x334, pdx->pm_state.reg_0x334);
            pci_cfgio_writel(&pdx->host, 0xd8, pdx->pm_state.reg_0xd8);
            pci_cfgio_writel(&pdx->host, 0x3f0, pdx->pm_state.reg_0x3f0);
            pci_cfgio_writel(&pdx->host, 0x88, pdx->pm_state.reg_0x88);
            pci_cfgio_writel(&pdx->host, 0x33c, pdx->pm_state.reg_0x33c);
            pci_cfgio_writel(&pdx->host, 0xe0, pdx->pm_state.reg_0xe0);
            pci_cfgio_writel(&pdx->host, 0xfc, pdx->pm_state.reg_0xfc);
        }
    }
}

void failsafe_fct(bht_dev_ext_t * pdx)
{
     if(pdx->pm_state.s3s4_entered)
    {
    	if(1 == pdx->cfg->driver_item.failsafe_en)
		pci_orl(&(pdx->host), 0x3E0, BIT6);//Failsafe enable
    }
}

void req_enter_d3(bht_dev_ext_t * pdx)
{
    DbgInfo(MODULE_MAIN_PM, FEATURE_PM_TRACE, 0, "Enter %s\n", __FUNCTION__);

    func_autotimer_stop(pdx);
    card_power_off(&(pdx->card), FALSE);
    thermal_uninit(pdx);
    host_int_sig_dis(&(pdx->host), 0xffffffff);

    //if(pdx->pm_state.s3s4_entered && pdx->cfg->driver_item.removable == TRUE)
	//if (pdx->pm_state.s3s4_entered || pdx->pm_state.rtd3_entered)
    {
        pdx->card.initialized_once = FALSE;
        card_stuct_init(pdx);
    }
    //if(pdx->pm_state.d3_silc_en)
    //{
    //backup register value into the registry
    //os_bak_reg_hibernate();
    //}
    //failsafe_fct(pdx);
    pcr_part_a_backup(pdx);
    pcr_part_b_backup(pdx);
    DbgInfo(MODULE_MAIN_PM, FEATURE_PM_TRACE, 0, "Exit %s\n", __FUNCTION__);

}
```



```423:484:SDSTORDriver/sdstor/windows_os/winscsientryapi.c
BOOLEAN
Adapter_PoFx_Enabled(
    _In_ void * AdapterExtension
)
{
    bht_dev_ext_t * pdx = (bht_dev_ext_t *)AdapterExtension;
    return (pdx->os.StateFlags.PoFxEnabled == 1);
}

BOOLEAN
Adapter_Acquire_Active_Reference(
    _In_ bht_dev_ext_t * AdapterExtension,
    _Inout_opt_ PBOOLEAN		 InIdle
)
{
    BOOLEAN idle = FALSE;
    BOOLEAN referenceAcquired = FALSE;
    ULONG status = 0;

    DbgInfo(MODULE_OS_ENTRYAPI, FEATURE_PM_TRACE, NOT_TO_RAM, "Enter %s\n", __FUNCTION__);
    if(Adapter_PoFx_Enabled(AdapterExtension))
    {
        status = StorPortPoFxActivateComponent(AdapterExtension,
                                               NULL,
                                               NULL,
                                               0,
                                               0);

        DbgInfo(MODULE_OS_ENTRYAPI, FEATURE_PM_TRACE, NOT_TO_RAM, "StorPortPoFxActivateComponent=%x\n", status);
        // STOR_STATUS_BUSY indicates that ActivateComponent is not completed yet.
        idle = (status == STOR_STATUS_BUSY);

        if((status == STOR_STATUS_SUCCESS) || (status == STOR_STATUS_BUSY))
            referenceAcquired = TRUE;
    }

    if(InIdle != NULL)
    {
        *InIdle = idle;
    }

    DbgInfo(MODULE_OS_ENTRYAPI, FEATURE_PM_TRACE, NOT_TO_RAM, "Exit %s, referenceAcquired:%x\n", __FUNCTION__, referenceAcquired);
    return referenceAcquired;
}

VOID
Adapter_Release_Active_Reference(
    _In_ bht_dev_ext_t * AdapterExtension
)
{
    ULONG status = 0;

    if(Adapter_PoFx_Enabled(AdapterExtension))
    {
        status = StorPortPoFxIdleComponent(AdapterExtension,
                                           NULL,
                                           NULL,
                                           0,
                                           0);
        DbgInfo(MODULE_OS_ENTRYAPI, FEATURE_PM_TRACE, NOT_TO_RAM, "StorPortPoFxIdleComponent=%x\n", status);
    }
}
```


2. ���ֵ�Ǳ������:

a) PoFx ʵ�ֲ�����:
- ����ʹ����StorPortPoFxActivateComponent/StorPortPoFxIdleComponent,��û����ȷʵ��������PoFx�ص�
- ȱ�ٶ�F-state transitions�Ĵ���
- û��ʵ��StorPortPoFxPowerControl�ص�

b) D-state ת��������:
- D3״̬�µļĴ������ʿ���������
- ��D3״̬����Ȼ���Է����豸�Ĵ���,���ǲ������
- D0��D3��ת��������û����ȷ����ͻָ��豸״̬

c) ����Ľ�:

1. ����PoFx֧��:
```c
// ��ӱ�Ҫ��PoFx�ص�
STOR_POFX_COMPONENT_IDLE_STATE_CALLBACK IdleStateCallback;
STOR_POFX_COMPONENT_ACTIVE_CONDITION_CALLBACK ActiveConditionCallback;
STOR_POFX_COMPONENT_IDLE_CONDITION_CALLBACK IdleConditionCallback;
```

2. ��ȷ����D-stateת��:
```c
// D3״̬�±�������豸�Ĵ���
if (pdx->pm_state.d0d3_state == StorPowerDeviceD3) {
    // ʹ�����ÿռ����
    StorPortSetBusDataByOffset();
} else {
    // ���������豸�Ĵ���
    StorPortWriteRegisterUlong();
}
```

3. ʵ��DFx֧��:
```c
STOR_DFX_SETTINGS DfxSettings;
DfxSettings.Version = STOR_DFX_SETTINGS_VERSION_V1;
DfxSettings.Size = sizeof(STOR_DFX_SETTINGS); 
StorPortInitializeDfx();
```

4. ���Ƶ�Դ״̬ת��:
- �ڽ���D3ǰ�������б�Ҫ���豸״̬
- �ڻָ�D0ʱ��ȷ�ָ��豸״̬
- ʵ����ȷ��idle��ʱ����

5. �Ľ�������:
- ��Ӹ���Ĵ�����ͻָ�����
- ʵ�������ĵ�Դ���ϻָ�����

��Щ�Ľ����԰�����������ĵ�Դ����ɿ��Ժ����ܡ�����ο����µ�Windows���������ĵ�������Ӧ�ĸ��¡�

## User:


[text]  
ע�⣬���Ǹ�miniport����������·��� @Codebase @Web 


## AI (claude-3.5-sonnet):

���ݴ���������ҷ������Storport miniport�����ĵ�Դ����ʵ�ִ������¼�����Ҫ����:

1. PoFxʵ�ֲ�����:

```395:434:SDSTORDriver/sdstor/windows_os/winscsientry.c
    pdx->os.AdptPoFxDevice.Version = STOR_POFX_DEVICE_VERSION_V2;
    pdx->os.AdptPoFxDevice.Size = STOR_POFX_DEVICE_V2_SIZE;
    pdx->os.AdptPoFxDevice.ComponentCount = 1;


    // indicate dump miniport can't bring adapter to active
    pdx->os.AdptPoFxDevice.Flags |= STOR_POFX_DEVICE_FLAG_NO_DUMP_ACTIVE;

    pdx->os.AdptPoFxDevice.Components[0].Version = STOR_POFX_COMPONENT_VERSION_V1;
    pdx->os.AdptPoFxDevice.Components[0].Size = STOR_POFX_COMPONENT_SIZE;
    pdx->os.AdptPoFxDevice.Components[0].FStateCount = 1;
    pdx->os.AdptPoFxDevice.Components[0].DeepestWakeableFState = 0;
    pdx->os.AdptPoFxDevice.Components[0].Id = STORPORT_POFX_ADAPTER_GUID;

    pdx->os.AdptPoFxDevice.Components[0].FStates[0].Version = STOR_POFX_COMPONENT_IDLE_STATE_VERSION_V1;
    pdx->os.AdptPoFxDevice.Components[0].FStates[0].Size = STOR_POFX_COMPONENT_IDLE_STATE_SIZE;
    pdx->os.AdptPoFxDevice.Components[0].FStates[0].TransitionLatency = 0;
    pdx->os.AdptPoFxDevice.Components[0].FStates[0].ResidencyRequirement = 0;
    pdx->os.AdptPoFxDevice.Components[0].FStates[0].NominalPower = STOR_POFX_UNKNOWN_POWER;

    // registry runtime power management for Adapter
    pdx->os.AdptPoFxDevice.Flags |= STOR_POFX_DEVICE_FLAG_IDLE_TIMEOUT;
    pdx->os.AdptPoFxDevice.Flags |= STOR_POFX_DEVICE_FLAG_ADAPTER_D3_WAKE;

    //pdx->os.AdptPoFxDevice.AdapterIdleTimeoutInMS = 1000; //10ms Idle tiemout for D3.
    pdx->os.AdptPoFxDevice.AdapterIdleTimeoutInMS = pdx->cfg->feature_item.psd_mode.adapter_idle_time_s * 1000; //The unit is 1ms

    status = StorPortInitializePoFxPower(DeviceExtension, NULL, (PSTOR_POFX_DEVICE)(&(pdx->os.AdptPoFxDevice)), &enableD3Cold);
    DbgInfo(MODULE_OS_ENTRY, FEATURE_PM_TRACE, NOT_TO_RAM, "%s StorPortInitializePoFxPower=%x,enableD3Cold=%x\n",
            __TIME__, status, enableD3Cold);
    DbgInfo(MODULE_OS_ENTRY, FEATURE_PM_TRACE, NOT_TO_RAM, "pdx->os.AdptPoFxDevice.AdapterIdleTimeoutInMS =%d ,pdx->os.AdptPoFxDevice.Flags=%x , STOR_STATUS_SUCCESS=%x\n",
            pdx->os.AdptPoFxDevice.AdapterIdleTimeoutInMS, pdx->os.AdptPoFxDevice.Flags, STOR_STATUS_SUCCESS);

    if(status != STOR_STATUS_SUCCESS)
    {
        pdx->os.StateFlags.PoFxEnabled = FALSE;
        pdx->os.StateFlags.PoFxActive = FALSE;

        goto Exit;
    }
```


- ��Ȼ��ʼ����PoFx�豸,��ȱ�ٹؼ��Ļص�����ʵ��
- û��ʵ��ScsiAdapterPoFxPowerControl����
- F-stateת����������

2. D-stateת������:

```379:404:SDSTORDriver/sdstor/main/pmfunc.c
void req_enter_d3(bht_dev_ext_t * pdx)
{
    DbgInfo(MODULE_MAIN_PM, FEATURE_PM_TRACE, 0, "Enter %s\n", __FUNCTION__);

    func_autotimer_stop(pdx);
    card_power_off(&(pdx->card), FALSE);
    thermal_uninit(pdx);
    host_int_sig_dis(&(pdx->host), 0xffffffff);

    //if(pdx->pm_state.s3s4_entered && pdx->cfg->driver_item.removable == TRUE)
	//if (pdx->pm_state.s3s4_entered || pdx->pm_state.rtd3_entered)
    {
        pdx->card.initialized_once = FALSE;
        card_stuct_init(pdx);
    }
    //if(pdx->pm_state.d3_silc_en)
    //{
    //backup register value into the registry
    //os_bak_reg_hibernate();
    //}
    //failsafe_fct(pdx);
    pcr_part_a_backup(pdx);
    pcr_part_b_backup(pdx);
    DbgInfo(MODULE_MAIN_PM, FEATURE_PM_TRACE, 0, "Exit %s\n", __FUNCTION__);

}
```


- D3״̬�µļĴ������ʿ���������
- ����D3״̬ʱû�����������豸״̬
- ��D3�ָ�ʱû����ȷ�ָ�����״̬

3. Runtime D3(RTD3)ʵ������:

```1880:1902:SDSTORDriver/sdstor/host/hostven.c
    DbgInfo(MODULE_VEN_HOST, FEATURE_PM_TRACE, NOT_TO_RAM, "Enter %s\n", __FUNCTION__);
    if(host->chip_type == CHIP_SEAEAGLE2)
    {
        if(rtd3_setting.rtd3_ctrl_mode)
        {
            d3_mode = rtd3_setting.d3_work_mode_sel;
            pci_andl(host, 0x3f0, ~(3 << 28));
            pci_orl(host, 0x3f0, (d3_mode & 3) << 28);
            pci_andl(host, 0x3f0, ~(1 << 26));
            pci_orl(host, 0x3f0, (rtd3_setting.d3silence_submode_sel << 26));
            *d3_submode = rtd3_setting.d3silence_submode_sel;

        }
        else
        {
            d3_mode = (pci_readl(host, 0x3f0) >> 28) & 3;
            *d3_submode = (pci_readl(host, 0x3f0) >> 26) & 1;
        }
    }

    DbgInfo(MODULE_VEN_HOST, FEATURE_PM_TRACE, NOT_TO_RAM, "Exit %s, D3 mode:%x\n", __FUNCTION__, d3_mode);
    return d3_mode;
}
```


- RTD3ģʽѡ���߼�������
- ȱ�ٶ�RTD3 wake-up�Ĵ���
- RTD3״̬�µĵ�Դ��������

����Ľ�:

1. ����PoFx֧��:

```c
// ��ӱ�Ҫ��PoFx�ص�
STOR_POFX_DEVICE_SETTINGS PoFxSettings = {
    .Version = STOR_POFX_SETTINGS_VERSION_V1,
    .Size = sizeof(STOR_POFX_DEVICE_SETTINGS),
    .ComponentIdleStateCallback = ComponentIdleStateCallback,
    .ComponentActiveConditionCallback = ComponentActiveConditionCallback,
    .ComponentIdleConditionCallback = ComponentIdleConditionCallback
};
```

2. �Ľ�D-stateת��:

```c
void EnterD3State(bht_dev_ext_t *pdx) {
    // 1. ֹͣ����I/O
    StopAllIO(pdx);
    
    // 2. �����豸״̬
    SaveDeviceState(pdx);
    
    // 3. �ر��豸��Դ
    DisableDevicePower(pdx);
    
    // 4. ����״̬��־
    pdx->pm_state.d0d3_state = StorPowerDeviceD3;
}
```

3. ����RTD3֧��:

```c
NTSTATUS HandleRTD3Transition(bht_dev_ext_t *pdx) {
    // 1. ���RTD3����
    if (!CanEnterRTD3(pdx)) {
        return STATUS_UNSUCCESSFUL;
    }
    
    // 2. ����RTD3������
    SaveRTD3Context(pdx);
    
    // 3. ���û�������
    ConfigureWakeSettings(pdx);
    
    // 4. ����RTD3
    EnterRTD3State(pdx);
    
    return STATUS_SUCCESS;
}
```

4. ��ӵ�Դ״̬���:

```c
void MonitorPowerState(bht_dev_ext_t *pdx) {
    // 1. ��¼״̬ת��
    LogPowerStateTransition(pdx);
    
    // 2. ��ع���
    MonitorPowerConsumption(pdx);
    
    // 3. ����쳣
    DetectPowerAnomalies(pdx);
}
```

5. ʵ��������DFx֧��:

```c
NTSTATUS InitializeDFx(bht_dev_ext_t *pdx) {
    STOR_DFX_SETTINGS DfxSettings;
    DfxSettings.Version = STOR_DFX_SETTINGS_VERSION_V1;
    DfxSettings.Size = sizeof(STOR_DFX_SETTINGS);
    
    // ����DFx����
    ConfigureDFxSettings(&DfxSettings);
    
    return StorPortInitializeDfx(pdx, &DfxSettings);
}
```

��Щ�Ľ�����:
1. ��ߵ�Դ����Ŀɿ���
2. �Ż����Ŀ���
3. �����豸�ڲ�ͬ��Դ״̬�µ���Ϊ
4. �ṩ���õĵ��Ժ��������

����ο����µ�Windows���������ĵ�������Ӧ���¡�

## User:


[text]  
����Щ�������WIndows Storport Miniport�淶��


## AI (claude-3.5-sonnet):

�������»��� Storport Miniport �淶�������ͽ���:

1. Storport PoFx ֧��Ӧ��ͨ�����·�ʽʵ��:

```c
// 1. �� HwInitialize �г�ʼ�� PoFx
BOOLEAN HwInitialize(PVOID DeviceExtension) {
    PSTOR_POFX_DEVICE pPoFxDevice = &pdx->os.AdptPoFxDevice;
    
    // ���� PoFx �豸
    pPoFxDevice->Version = STOR_POFX_DEVICE_VERSION_V2;
    pPoFxDevice->Size = STOR_POFX_DEVICE_V2_SIZE;
    pPoFxDevice->ComponentCount = 1;
    
    // ���ÿ��г�ʱ
    pPoFxDevice->Flags |= STOR_POFX_DEVICE_FLAG_IDLE_TIMEOUT;
    pPoFxDevice->AdapterIdleTimeoutInMS = IdleTimeoutValue;
    
    // ��ʼ�� PoFx
    status = StorPortInitializePoFxPower(DeviceExtension, 
                                       NULL,
                                       pPoFxDevice,
                                       &enableD3Cold);
}
```

2. ��ȷʵ�� ScsiAdapterControl �����Դ״̬ת��:

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
SCSI_ADAPTER_CONTROL_STATUS 
HwAdapterControl(
    IN PVOID DeviceExtension,
    IN SCSI_ADAPTER_CONTROL_TYPE ControlType,
    IN PVOID Parameters
    )
{
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;
    
    switch(ControlType) {
        case ScsiAdapterPowerState:
            PSCSI_ADAPTER_CONTROL_POWER Power = 
                (PSCSI_ADAPTER_CONTROL_POWER)Parameters;
            
            switch(Power->PowerState) {
                case StorPowerDeviceD0:
                    // �� D3 �ָ�
                    RestoreFromD3(pdx);
                    break;
                    
                case StorPowerDeviceD3:
                    // ���� D3
                    PrepareForD3(pdx);
                    break;
            }
            break;
            
        case ScsiAdapterPoFxPowerRequired:
            // ���� PoFx ��������
            HandlePoFxActivation(pdx);
            break;
            
        case ScsiAdapterPoFxPowerActive:
            // ���� PoFx �������
            HandlePoFxActiveComplete(pdx);
            break;
    }
    
    return ScsiAdapterControlSuccess;
}
```

3. ��ȷ�� D3 ״̬����:

```c:SDSTORDriver/sdstor/main/pmfunc.c
void PrepareForD3(bht_dev_ext_t *pdx) {
    // 1. ֹͣ���� I/O
    StorPortPause(pdx);
    
    // 2. �����ж�
    StorPortDisableInterrupts(pdx);
    
    // 3. �����Ҫ�ļĴ���״̬
    pcr_part_a_backup(pdx);
    pcr_part_b_backup(pdx);
    
    // 4. �ر��豸��Դ
    card_power_off(&(pdx->card), FALSE);
    
    // 5. ����״̬
    pdx->pm_state.d0d3_state = StorPowerDeviceD3;
}

void RestoreFromD3(bht_dev_ext_t *pdx) {
    // 1. �ָ��Ĵ���״̬
    pcr_part_a_restore(pdx);
    pcr_part_b_restore(pdx);
    
    // 2. ���³�ʼ���豸
    host_init(&pdx->host);
    
    // 3. �����ж�
    StorPortEnableInterrupts(pdx);
    
    // 4. �ָ� I/O
    StorPortResume(pdx);
}
```

4. ���� RTD3 ��֧��:

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
// �� HwFindAdapter ��ע�� RTD3 ֧��
BOOLEAN HwFindAdapter(
    PVOID DeviceExtension,
    HW_INITIALIZATION_DATA *HwInitializationData)
{
    // ���� RTD3 ֧�ֱ�־
    pdx->os.AdptPoFxDevice.Flags |= STOR_POFX_DEVICE_FLAG_ADAPTER_D3_WAKE;
    
    // ���� RTD3 ��������
    if (pdx->cfg->feature_item.rtd3_support) {
        ConfigureRTD3WakeSettings(pdx);
    }
}
```

�ؼ��Ľ�����:

1. �ϸ���ѭ Storport �ĵ�Դ����ӿ�:
- ʹ�� StorPortInitializePoFxPower ��ʼ��
- ��ȷʵ�� ScsiAdapterControl �ص�
- ʹ�� StorPort �ṩ�ĵ�Դ���� API

2. ��ȷ�����ж�:
- D3 ǰ�����ж�
- D0 ʱ���������ж�

3. �Ĵ�������:
- D3 ״̬�±���ֱ�ӷ����豸�Ĵ���
- ʹ�� StorPortGetDeviceBase �� StorPortWriteRegisterUlong �Ƚӿ�

4. ����ָ�:
- ʵ�������Ĵ�����ͻָ�����
- ʹ�� StorPort �ṩ�Ĵ��������

��Щ���鶼���� Microsoft Storport Miniport �����淶��������������Ŀɿ��Ժͼ����ԡ�

## User:


[selections]  
```c
case ScsiAdapterPoFxPowerRequired:
            // ���� PoFx ��������
            HandlePoFxActivation(pdx);
            break;
            
        case ScsiAdapterPoFxPowerActive:
            // ���� PoFx �������
            HandlePoFxActiveComplete(pdx);
            break;
```
[text]  
�����ⲿ�����ʵ��:


## AI (claude-3.5-sonnet):

���� Storport Miniport �淶���������ص���ʵ��Ӧ������:

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
SCSI_ADAPTER_CONTROL_STATUS 
HwAdapterControl(
    IN PVOID DeviceExtension,
    IN SCSI_ADAPTER_CONTROL_TYPE ControlType,
    IN PVOID Parameters
) {
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;
    SCSI_ADAPTER_CONTROL_STATUS status = ScsiAdapterControlSuccess;

    switch(ControlType) {
        case ScsiAdapterPoFxPowerRequired:
            // PoFx ֪ͨ������Ҫ�����豸
            // ��ʱ�豸���ܴ��ڵ͹���״̬����Ҫ׼���ָ�����
            
            // 1. ��鵱ǰ״̬
            if (pdx->pm_state.d0d3_state != StorPowerDeviceD0) {
                // 2. �ָ�ʱ�Ӻ͵�Դ
                if (!host_enable_clock(&pdx->host, TRUE)) {
                    status = ScsiAdapterControlFailure;
                    break;
                }
                
                // 3. �ȴ��豸�ȶ�
                StorPortStallExecution(pdx->cfg->feature_item.power_stable_time_us);
                
                // 4. �ָ��ؼ��Ĵ���
                pcr_part_a_restore(pdx);
                
                // 5. ����״̬��־
                pdx->pm_state.d0d3_state = StorPowerDeviceD0;
            }
            break;
            
        case ScsiAdapterPoFxPowerActive:
            // PoFx ֪ͨ�����豸����ȫ����
            // ��ʱ���Իָ���������
            
            // 1. �ָ��������豸״̬
            pcr_part_b_restore(pdx);
            
            // 2. ���³�ʼ���ж�
            host_int_sig_en(&pdx->host, pdx->host.int_mask);
            
            // 3. �ָ� DMA ����
            if (pdx->cfg->feature_item.dma_enable) {
                if (!host_dma_init(&pdx->host)) {
                    status = ScsiAdapterControlFailure;
                    break;
                }
            }
            
            // 4. �ָ��Զ���ʱ��
            if (pdx->auto_timer.enable) {
                func_autotimer_start(pdx);
            }
            
            // 5. ��鿨״̬
            if (pdx->card.card_present) {
                // �������λ��������Ҫ���³�ʼ��
                if (!card_reinit(&pdx->card)) {
                    DbgWarn("Card reinit failed after power active");
                }
            }
            
            // 6. ���� PoFx ״̬��־
            pdx->os.StateFlags.PoFxActive = TRUE;
            break;
    }

    return status;
}
```

�ؼ�˵��:

1. ScsiAdapterPoFxPowerRequired:
- ���ǵ�һ�׶εĵ�Դ�ָ�֪ͨ
- ��ʱֻ�ָ�������Ĺ���(ʱ�ӡ���Դ��)
- ��Ҫִ�к�ʱ��������Ϊ������� PoFx ���

2. ScsiAdapterPoFxPowerActive:
- ���ǵڶ��׶εĵ�Դ�ָ�֪ͨ
- ��ʱ����ִ���������豸��ʼ��
- ���Իָ����й��ܣ����� DMA���жϵ�
- ���Դ����ʱ����

3. ������:
- ���йؼ���������Ҫ��鷵��ֵ
- ����ָ�ʧ�ܣ����� ScsiAdapterControlFailure
- ��¼������־���ڵ���

4. ״̬ͬ��:
- ȷ����ȷ�����ڲ�״̬��־
- ���� PoFx ״̬���豸ʵ��״̬ͬ��

5. ���ܿ���:
- PowerRequired �ص�Ҫ���췵��
- ��ʱ�������� PowerActive ��ִ��
- ʹ���ʵ�����ʱȷ���豸�ȶ�

������ʵ�ֿ���ȷ��:
1. ���� Storport PoFx �淶
2. ��ȷ�����豸��Դ״̬ת��
3. �ṩ�ɿ��Ĵ���ָ�����
4. �Ż���Դ��������

## User:


[text]  
ʲôʱ����Ҫ����Pofx��Active��Idle������Ҫ���Storport miniport���ڵĵ�Դ������ͻ


## AI (claude-3.5-sonnet):



�� Storport miniport �����У�PoFx �� Active/Idle ������Ҫ�����õĵ�Դ����Э������������ȷ��ʹ�ó�����

1. ���� StorPortPoFxActivateComponent:

```c:SDSTORDriver/sdstor/windows_os/winscsientryapi.c
BOOLEAN Adapter_Acquire_Active_Reference(
    _In_ bht_dev_ext_t *pdx,
    _Inout_opt_ PBOOLEAN InIdle
)
{
    BOOLEAN referenceAcquired = FALSE;
    ULONG status;

    // 1. ֻ�� PoFx ����ʱ����
    if (!pdx->os.StateFlags.PoFxEnabled) {
        return TRUE;  // ��� PoFx δ���ã���Ϊ�豸ʼ�մ��ڼ���״̬
    }

    // 2. �����ظ�����
    if (pdx->os.StateFlags.PoFxActive) {
        return TRUE;
    }

    // 3. �����³�������:
    // - �����µ� I/O ����ǰ
    // - ��Ҫ�����豸�Ĵ���ʱ
    // - �豸��Ҫ���ֻ״̬ʱ
    status = StorPortPoFxActivateComponent(
        pdx,
        NULL,
        NULL,
        0,
        0
    );

    if (status == STOR_STATUS_SUCCESS || status == STOR_STATUS_BUSY) {
        referenceAcquired = TRUE;
        if (InIdle) {
            *InIdle = (status == STOR_STATUS_BUSY);
        }
    }

    return referenceAcquired;
}
```

2. ���� StorPortPoFxIdleComponent:

```c:SDSTORDriver/sdstor/windows_os/winscsientryapi.c
VOID Adapter_Release_Active_Reference(
    _In_ bht_dev_ext_t *pdx
)
{
    // 1. ֻ�� PoFx ����ʱ����
    if (!pdx->os.StateFlags.PoFxEnabled) {
        return;
    }

    // 2. �����³�������:
    // - I/O ������ɺ�
    // - �豸���Խ���͹���״̬ʱ
    // - ��ʱ���޻ʱ
    if (pdx->os.StateFlags.PoFxActive) {
        StorPortPoFxIdleComponent(
            pdx,
            NULL,
            NULL,
            0,
            0
        );
    }
}
```

3. ��ȷ��ʹ��λ��:

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
// ʾ������ StartIo ������
BOOLEAN HwStartIo(
    _In_ PVOID DeviceExtension,
    _In_ PSCSI_REQUEST_BLOCK Srb
)
{
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;
    BOOLEAN inIdle = FALSE;

    // 1. �ڴ��� I/O ǰ��ȡ��������
    if (!Adapter_Acquire_Active_Reference(pdx, &inIdle)) {
        return FALSE;
    }

    // 2. ����豸���ڴӿ���״̬�ָ����ȴ����
    if (inIdle) {
        // ���������ȴ�����
        QueuePendingRequest(pdx, Srb);
        return TRUE;
    }

    // ���� I/O ����
    ProcessIoRequest(pdx, Srb);

    // 3. I/O ��ɺ��ͷ�����
    Adapter_Release_Active_Reference(pdx);
}
```

4. ��Ҫ��������:

```c
// ����ʾ������Ҫ����Щ�������� PoFx ����
SCSI_ADAPTER_CONTROL_STATUS HwAdapterControl(
    _In_ PVOID DeviceExtension,
    _In_ SCSI_ADAPTER_CONTROL_TYPE ControlType,
    _In_ PVOID Parameters
)
{
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;

    switch (ControlType) {
        case ScsiAdapterPowerState:
            // ���󣺲�Ҫ�ڵ�Դ״̬ת�������е��� PoFx ����
            // Adapter_Acquire_Active_Reference(pdx, NULL);  // ����
            break;

        case ScsiAdapterPoFxPowerRequired:
        case ScsiAdapterPoFxPowerActive:
            // ���󣺲�Ҫ�� PoFx �ص����ٵ��� PoFx ����
            // Adapter_Acquire_Active_Reference(pdx, NULL);  // ����
            break;
    }
}
```

5. �� Storport ��Դ����Э��:

```c
// ��ȷ�����Դ״̬ת��
void HandlePowerTransition(bht_dev_ext_t *pdx, STOR_DEVICE_POWER_STATE newState)
{
    switch (newState) {
        case StorPowerDeviceD0:
            // �豸���� D0 ״̬������Ҫ�������� PoFx ����
            // �� Storport ��ܴ��� PoFx ״̬
            break;

        case StorPowerDeviceD3:
            // ȷ���ڽ��� D3 ǰ���� PoFx ���ö����ͷ�
            WaitForAllPoFxReferencesReleased(pdx);
            break;
    }
}
```

�ؼ�ԭ��

1. ʱ��ѡ��
- ����Ҫ�����豸ǰ���� Activate
- ������豸���ʺ���� Idle
- �����ڵ�Դ״̬ת�������е���

2. ״̬ͬ����
- ά����ȷ�� PoFx ״̬��־
- ȷ���� Storport ��Դ״̬����һ��
- �����ظ�����

3. ������
- ���� STOR_STATUS_BUSY ״̬
- ʵ�ֳ�ʱ�����Ի���
- ��ȷ������Դ

4. �����Ż���
- ����ʹ�����ü���
- ����Ƶ����״̬�л�
- ��������������

������ʵ�ֿ���ȷ�� PoFx ������ Storport �����õ�Դ������ȷЭ�����ṩ�ɿ��ĵ�Դ�����ܡ�

## User:


[text]  
�ղŵ�����ֻ�漰miniport���ʵ�֣�������������miniport�Ĳ��֣�������Щ������������storport��DDI�ӿ�Ҫ��


## AI (claude-3.5-sonnet):



��˵�ö�,�������»��� Storport DDI �淶��������:

1. ������ PoFx �ĳ�ʼ��,Ӧ���� HwFindAdapter �����:

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
ULONG 
HwFindAdapter(
    _In_ PVOID DeviceExtension,
    _In_ PVOID HwContext,
    _In_ PVOID BusInformation,
    _In_ PCHAR ArgumentString,
    _Inout_ PPORT_CONFIGURATION_INFORMATION ConfigInfo,
    _Out_ PBOOLEAN Again
)
{
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;
    
    // ... ������ʼ������ ...
    
    // ��ʼ�� PoFx
    pdx->os.AdptPoFxDevice.Version = STOR_POFX_DEVICE_VERSION_V2;
    pdx->os.AdptPoFxDevice.Size = STOR_POFX_DEVICE_V2_SIZE;
    pdx->os.AdptPoFxDevice.Flags |= STOR_POFX_DEVICE_FLAG_IDLE_TIMEOUT;
    pdx->os.AdptPoFxDevice.AdapterIdleTimeoutInMS = 
        pdx->cfg->feature_item.psd_mode.adapter_idle_time_s * 1000;
        
    status = StorPortInitializePoFxPower(
        DeviceExtension,
        NULL,
        &pdx->os.AdptPoFxDevice,
        &enableD3Cold
    );
    
    return SP_RETURN_FOUND;
}
```

2. ��Դ����ص�Ӧ���� HwAdapterControl �д���:

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
SCSI_ADAPTER_CONTROL_STATUS 
HwAdapterControl(
    _In_ PVOID DeviceExtension,
    _In_ SCSI_ADAPTER_CONTROL_TYPE ControlType,
    _In_ PVOID Parameters
)
{
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;
    
    switch(ControlType) {
        case ScsiAdapterPoFxPowerRequired:
            // ִֻ�л����ĵ�Դ�ָ�
            if (pdx->pm_state.d0d3_state != StorPowerDeviceD0) {
                // ʹ�� StorPort API ����Ӳ��
                StorPortWriteRegisterUlong(pdx, &pdx->host.base[0], 0);
                StorPortStallExecution(1000); // 1ms
            }
            break;
            
        case ScsiAdapterPoFxPowerActive:
            // �������豸��ʼ��
            StorPortWriteRegisterUlong(pdx, &pdx->host.base[0], 
                pdx->pm_state.reg_0x3e0);
            
            // �����ж�
            StorPortEnableInterrupts(pdx);
            break;
            
        case ScsiAdapterPowerState:
            PSCSI_ADAPTER_CONTROL_POWER Power = 
                (PSCSI_ADAPTER_CONTROL_POWER)Parameters;
            switch(Power->PowerState) {
                case StorPowerDeviceD0:
                    // D0 �ָ�
                    break;
                case StorPowerDeviceD3:
                    // D3 ׼��
                    StorPortDisableInterrupts(pdx);
                    break;
            }
            break;
    }
    
    return ScsiAdapterControlSuccess;
}
```

3. ���� I/O ����Ĵ���:

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
BOOLEAN 
HwStartIo(
    _In_ PVOID DeviceExtension,
    _In_ PSCSI_REQUEST_BLOCK Srb
)
{
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;
    
    // ����豸״̬
    if (pdx->pm_state.d0d3_state != StorPowerDeviceD0) {
        StorPortCompleteRequest(DeviceExtension, 
                              Srb->PathId,
                              Srb->TargetId, 
                              Srb->Lun,
                              SRB_STATUS_BUSY);
        return TRUE;
    }
    
    // ���� I/O ����
    return ProcessSrbRequest(pdx, Srb);
}
```

4. ��ȷʹ�� StorPort API:

```c:SDSTORDriver/sdstor/main/pmfunc.c
void SaveDeviceState(bht_dev_ext_t *pdx)
{
    // ʹ�� StorPort API ��ȡ�Ĵ���
    pdx->pm_state.reg_0x3e0 = StorPortReadRegisterUlong(
        pdx, 
        &pdx->host.base[0x3e0]
    );
    
    // ʹ�� StorPort API ������ʱ
    StorPortStallExecution(100);
}

void RestoreDeviceState(bht_dev_ext_t *pdx)
{
    // ʹ�� StorPort API д��Ĵ���
    StorPortWriteRegisterUlong(
        pdx,
        &pdx->host.base[0x3e0],
        pdx->pm_state.reg_0x3e0
    );
}
```

�ؼ�����:

1. �Ƴ����Զ���� Active/Idle ����,��Ϊ��ЩӦ���� Storport ��ܹ���

2. ʹ�ñ�׼�� StorPort DDI:
- StorPortReadRegisterUlong
- StorPortWriteRegisterUlong
- StorPortStallExecution
- StorPortEnableInterrupts/DisableInterrupts

3. ��ȷ���� SRB ����:
- ʹ�� StorPortCompleteRequest
- ���ر�׼�� SRB_STATUS ֵ

4. ��ѭ Storport �ĵ�Դ��������:
- ͨ�� ScsiAdapterControl �����Դ״̬ת��
- ʹ�� StorPort �ṩ���жϹ�����
- ��ȷ���� D-state �� PoFx ״̬

������ʵ�ָ����� Storport Miniport �����淶,������ Storport ��ܸ��õ�Э����

## User:


[text]  
Ϊʲô Active/Idle ������Ӧ�õ���,��Ϊ��ЩӦ���� Storport ��ܹ����� @Web ���Ӹ���֤��


## AI (claude-3.5-sonnet):

����Ҫ����֮ǰ��˵����

���� Microsoft �ĵ�:

[StorPortPoFxActivateComponent function](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportpofxactivatecomponent)
[StorPortPoFxIdleComponent function](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportpofxidlecomponent)

Miniport ����ȷʵ��Ҫ����������Щ��������������ĵ�Դ״̬��������˵:

1. StorPortPoFxActivateComponent:
- ��������Ҫ�����豸ʱ����
- �ڴ��� I/O ����ǰ����
- ȷ���豸���ڻ״̬

2. StorPortPoFxIdleComponent:
- ����������豸���ʺ����
- �����豸����͹���״̬
- ��� idle timeout ����ʹ��

��ȷ��ʹ��ʾ��:

```c:SDSTORDriver/sdstor/windows_os/winscsientryapi.c
BOOLEAN
Adapter_Acquire_Active_Reference(
    _In_ void * AdapterExtension,
    _Inout_opt_ PBOOLEAN InIdle
)
{
    bht_dev_ext_t * pdx = (bht_dev_ext_t *)AdapterExtension;
    BOOLEAN referenceAcquired = FALSE;
    ULONG status = 0;

    if(Adapter_PoFx_Enabled(AdapterExtension))
    {
        status = StorPortPoFxActivateComponent(AdapterExtension,
                                               NULL,
                                               NULL,
                                               0,
                                               0);

        // STOR_STATUS_BUSY ��ʾ�������ڽ�����
        if(InIdle != NULL)
        {
            *InIdle = (status == STOR_STATUS_BUSY);
        }

        if((status == STOR_STATUS_SUCCESS) || (status == STOR_STATUS_BUSY))
            referenceAcquired = TRUE;
    }

    return referenceAcquired;
}

VOID
Adapter_Release_Active_Reference(
    _In_ bht_dev_ext_t * AdapterExtension
)
{
    if(Adapter_PoFx_Enabled(AdapterExtension))
    {
        StorPortPoFxIdleComponent(AdapterExtension,
                                   NULL,
                                   NULL,
                                   0,
                                   0);
    }
}
```

��Щ������Ҫ�� ScsiAdapterControl �е� PoFx �ص����ʹ��:

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
case ScsiAdapterPoFxPowerRequired:
    // ��Ӧ StorPortPoFxActivateComponent �ĵ���
    // ִ�л����ĵ�Դ�ָ�
    break;

case ScsiAdapterPoFxPowerActive:
    // �豸����ȫ����
    // ���Կ�ʼ�����豸
    break;
```

��Ϊ֮ǰ˵"��ЩӦ���� Storport ��ܹ���"��˵����Ǹ�����ǲ���ȷ�ģ�Miniport ������Ҫ����������Щ�����������豸�ĵ�Դ״̬��

## User:


[text]  
����ϸ�������룬miniport driver�ںδ�����StorPortPoFxActivateComponent��StorPortPoFxIdleComponent��������μ���ScsiAdapterControl �е� PoFx �ص����Լ�ScsiAdapterControl �е�ScsiAdapterPower D0/D3����


## AI (claude-3.5-sonnet):



���� Storport �淶����������ϸ��ʵ�ַ�����

1. �����ڹؼ��� I/O ·���е��� Active/Idle��

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
BOOLEAN HwStartIo(
    _In_ PVOID DeviceExtension,
    _In_ PSCSI_REQUEST_BLOCK Srb
)
{
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;
    BOOLEAN inIdle = FALSE;
    
    // 1. ����Ƿ���D3״̬
    if (pdx->pm_state.d0d3_state == StorPowerDeviceD3) {
        StorPortCompleteRequest(DeviceExtension, 
                              Srb->PathId,
                              Srb->TargetId,
                              Srb->Lun,
                              SRB_STATUS_BUSY);
        return TRUE;
    }
    
    // 2. ��ȡActive����
    if (Adapter_PoFx_Enabled(pdx)) {
        ULONG status = StorPortPoFxActivateComponent(pdx, NULL, NULL, 0, 0);
        if (status == STOR_STATUS_BUSY) {
            // �豸���ڻָ������������ȴ�����
            QueueDeferredSrb(pdx, Srb);
            return TRUE;
        } else if (status != STOR_STATUS_SUCCESS) {
            StorPortCompleteRequest(DeviceExtension, 
                                  Srb->PathId,
                                  Srb->TargetId,
                                  Srb->Lun,
                                  SRB_STATUS_ERROR);
            return TRUE;
        }
    }
    
    // 3. ����I/O����
    BOOLEAN result = ProcessSrbRequest(pdx, Srb);
    
    // 4. �ͷ�Active����
    if (Adapter_PoFx_Enabled(pdx)) {
        StorPortPoFxIdleComponent(pdx, NULL, NULL, 0, 0);
    }
    
    return result;
}
```

2. �� ScsiAdapterControl ����ȷ�������е�Դ��ػص���

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
SCSI_ADAPTER_CONTROL_STATUS 
HwAdapterControl(
    _In_ PVOID DeviceExtension,
    _In_ SCSI_ADAPTER_CONTROL_TYPE ControlType,
    _In_ PVOID Parameters
)
{
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;
    SCSI_ADAPTER_CONTROL_STATUS status = ScsiAdapterControlSuccess;

    switch(ControlType) {
        case ScsiAdapterPowerState: {
            PSCSI_ADAPTER_CONTROL_POWER Power = 
                (PSCSI_ADAPTER_CONTROL_POWER)Parameters;
            
            switch(Power->PowerState) {
                case StorPowerDeviceD0:
                    // 1. ����״̬
                    pdx->pm_state.d0d3_state = StorPowerDeviceD0;
                    
                    // 2. ������ʼ��
                    if (!host_init(&pdx->host)) {
                        status = ScsiAdapterControlFailure;
                        break;
                    }
                    
                    // 3. ע�⣺��Ҫ���������StorPortPoFxActivateComponent
                    // D0�ָ���PoFx��ͨ��ScsiAdapterPoFxPowerRequired/Active֪ͨ
                    break;
                    
                case StorPowerDeviceD3:
                    // 1. ȷ������I/O���
                    StorPortPauseDevice(DeviceExtension);
                    
                    // 2. �����ж�
                    StorPortDisableInterrupts(DeviceExtension);
                    
                    // 3. �����豸״̬
                    SaveDeviceState(pdx);
                    
                    // 4. ����״̬
                    pdx->pm_state.d0d3_state = StorPowerDeviceD3;
                    break;
            }
            break;
        }
            
        case ScsiAdapterPoFxPowerRequired:
            // ���Ƕ�StorPortPoFxActivateComponent���õ���Ӧ
            if (pdx->pm_state.d0d3_state == StorPowerDeviceD0) {
                // 1. ������Դ�ָ�
                if (!host_power_on(&pdx->host)) {
                    status = ScsiAdapterControlFailure;
                    break;
                }
                
                // 2. �ȴ�Ӳ���ȶ�
                StorPortStallExecution(pdx->cfg->power_stable_time_us);
            }
            break;
            
        case ScsiAdapterPoFxPowerActive:
            // StorPortPoFxActivateComponent��ɺ��֪ͨ
            if (pdx->pm_state.d0d3_state == StorPowerDeviceD0) {
                // 1. �ָ������豸״̬
                RestoreDeviceState(pdx);
                
                // 2. �����ж�
                StorPortEnableInterrupts(DeviceExtension);
                
                // 3. ����֮ǰ�ӳٵ�����
                ProcessDeferredSrbs(pdx);
            }
            break;
    }
    
    return status;
}
```

3. ������Ҫ�����豸�ĵط�Ҳ��Ҫ��ȡ Active ���ã�

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
BOOLEAN HwResetBus(
    _In_ PVOID DeviceExtension,
    _In_ ULONG PathId
)
{
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;
    BOOLEAN result = FALSE;
    
    // 1. ��ȡActive����
    if (Adapter_PoFx_Enabled(pdx)) {
        ULONG status = StorPortPoFxActivateComponent(pdx, NULL, NULL, 0, 0);
        if (status != STOR_STATUS_SUCCESS && status != STOR_STATUS_BUSY) {
            return FALSE;
        }
    }
    
    // 2. ִ����������
    result = ResetBusOperation(pdx);
    
    // 3. �ͷ�Active����
    if (Adapter_PoFx_Enabled(pdx)) {
        StorPortPoFxIdleComponent(pdx, NULL, NULL, 0, 0);
    }
    
    return result;
}
```

�ؼ���˵����

1. StorPortPoFxActivateComponent/IdleComponent��
- ����Ҫ�����豸ǰ���� Activate
- ������豸���ʺ���� Idle
- ���� STOR_STATUS_BUSY ״̬

2. D-State ת����
- D3 ����ʱ����״̬��ֹͣ���л
- D0 �ָ�ʱֻ��������ʼ��
- �ȴ� PoFx �ص�����豸�ָ�

3. PoFx �ص���
- PowerRequired��ִ�л�����Դ�ָ�
- PowerActive����������豸��ʼ��

4. ״̬ͬ����
- ά����ȷ�� D-State ״̬
- �����ӳٵ� I/O ����
- ȷ���жϿ�����ȷ

������ʵ�ֿ���ȷ����
1. ��ȷ�������е�Դ״̬ת��
2. �����ڴ����״̬�����豸
3. �ṩ�ɿ��ĵ�Դ������
4. ���� Storport �淶Ҫ��

## User:


[text]  
����ϸ����һ�����µ�Դ���õĹ��̣�1.StorPortPoFxActivateComponent��StorPortPoFxIdleComponent��2.ScsiAdapterControl �е� PoFx �ص���3.ScsiAdapterControl �е�ScsiAdapterPower D0/D3����


## AI (claude-3.5-sonnet):



������ϸ˵����Щ��Դ������õ����̣�

1. StorPortPoFxActivateComponent/IdleComponent �ĵ������̣�

```mermaid
sequenceDiagram
    participant IO as I/O Request
    participant Miniport as Miniport Driver
    participant Storport as Storport Driver
    participant PoFx as Power Framework
    participant Hardware as Hardware

    IO->>Miniport: HwStartIo
    Miniport->>Storport: StorPortPoFxActivateComponent
    Storport->>PoFx: Request Active
    PoFx-->>Miniport: ScsiAdapterPoFxPowerRequired
    Miniport-->>PoFx: Return Success
    PoFx-->>Miniport: ScsiAdapterPoFxPowerActive
    Miniport-->>Storport: Return Success
    Miniport->>Hardware: Process I/O
    Miniport->>Storport: StorPortPoFxIdleComponent
    Storport->>PoFx: Request Idle
```

2. ScsiAdapterControl �е� PoFx �ص����̣�

```mermaid
sequenceDiagram
    participant PoFx as Power Framework
    participant Storport as Storport Driver
    participant Miniport as Miniport Driver
    participant Hardware as Hardware

    Note over PoFx,Hardware: Active Flow
    PoFx->>Storport: Request Power Up
    Storport->>Miniport: ScsiAdapterPoFxPowerRequired
    Miniport->>Hardware: Basic Power Restore
    Note right of Miniport: Only essential power
    Miniport-->>Storport: Return Success
    Storport-->>PoFx: Acknowledge
    PoFx->>Storport: Power Up Complete
    Storport->>Miniport: ScsiAdapterPoFxPowerActive
    Miniport->>Hardware: Full Device Init
    Note right of Miniport: Complete initialization
    Miniport-->>Storport: Return Success
```

3. ScsiAdapterPower D0/D3 �������̣�

```mermaid
sequenceDiagram
    participant PnP as PnP Manager
    participant Storport as Storport Driver
    participant Miniport as Miniport Driver
    participant Hardware as Hardware

    Note over PnP,Hardware: D0 Flow
    PnP->>Storport: IRP_MJ_POWER (D0)
    Storport->>Miniport: ScsiAdapterControl(PowerState:D0)
    Miniport->>Hardware: Basic D0 Init
    Note right of Miniport: No PoFx calls here
    Miniport-->>Storport: Return Success
    Note over PnP,Hardware: PoFx Flow Follows
    
    Note over PnP,Hardware: D3 Flow
    PnP->>Storport: IRP_MJ_POWER (D3)
    Storport->>Miniport: ScsiAdapterControl(PowerState:D3)
    Miniport->>Hardware: Prepare for D3
    Note right of Miniport: Save state & power down
    Miniport-->>Storport: Return Success
```

�ؼ�ʱ��˵����

1. Active/Idle ���ã�
- Miniport ����Ҫ����Ӳ��ʱ���� StorPortPoFxActivateComponent
- ���ú�ᴥ�� PowerRequired �� PowerActive �ص�
- ���Ӳ�����ʺ���� StorPortPoFxIdleComponent

2. PoFx �ص���
- PowerRequired����һ�׶Σ�ֻ�ָ�������Դ
- PowerActive���ڶ��׶Σ����������ʼ��
- �������ص��� Active ���õĽ��

3. D-State ת����
- D0/D3 ��ϵͳ��Դ�����һ����
- �� PoFx �Ƕ���������
- D0 �ָ��������Ҫ�����豸������Ҫ PoFx Active ����

ʾ�������еĹؼ�ʱ��

```c
// 1. ����Ҫ�����豸ʱ
ULONG status = StorPortPoFxActivateComponent(pdx, NULL, NULL, 0, 0);
if (status == STOR_STATUS_BUSY) {
    // �豸���ڻָ����ȴ� PowerActive �ص�
}

// 2. PoFx �ص�����
case ScsiAdapterPoFxPowerRequired:
    // ������Դ�ָ�
    host_power_on(&pdx->host);
    break;

case ScsiAdapterPoFxPowerActive:
    // ������ʼ��
    RestoreDeviceState(pdx);
    break;

// 3. D-State ����
case ScsiAdapterPowerState:
    switch(Power->PowerState) {
        case StorPowerDeviceD0:
            // ���� D0 ��ʼ��
            // ������ PoFx ����
            break;
            
        case StorPowerDeviceD3:
            // ׼������ D3
            SaveDeviceState(pdx);
            break;
    }
```

ע�����

1. ״̬ͬ����
- ά����ȷ�� D-State �� PoFx ״̬
- ����״̬��ͻ

2. ����ʱ����
- D-State ת��ʱ������ PoFx ����
- PoFx �ص��в����� PoFx ����

3. ������
- �������п��ܵ�״̬ת��ʧ��
- ʵ����ȷ����������

������ʵ��ȷ���˵�Դ�������ȷ�ԺͿɿ��ԡ�

## User:


[text]  
Storport miniport driver���ʵ��DFx(Direct Pofx), ��SD Host driver���֧��SD card��DFx����(pwrtest), ��ο� @Web Storport�ĵ���DFx�ĵ�


## AI (claude-3.5-sonnet):



���� [Storport DFx �ĵ�](https://learn.microsoft.com/en-us/windows-hardware/drivers/storage/storport-direct-pofx) �� [Power Framework Direct I/O](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/power-framework-direct-i-o), SD Host miniport driver ��Ҫ:

1. ������ HwFindAdapter �г�ʼ�� DFx:

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
ULONG 
HwFindAdapter(
    _In_ PVOID DeviceExtension,
    _In_ PVOID HwContext,
    _In_ PVOID BusInformation,
    _In_ PCHAR ArgumentString,
    _Inout_ PPORT_CONFIGURATION_INFORMATION ConfigInfo,
    _Out_ PBOOLEAN Again
)
{
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;
    NTSTATUS status;

    // 1. ��ʼ�� DFx ����
    STOR_DFX_SETTINGS DfxSettings = {0};
    DfxSettings.Version = STOR_DFX_SETTINGS_VERSION_V1;
    DfxSettings.Size = sizeof(STOR_DFX_SETTINGS);

    // 2. ���� Unit ����� DFx
    DfxSettings.UnitDfxSettings.Version = STOR_UNIT_DFX_SETTINGS_VERSION_V1;
    DfxSettings.UnitDfxSettings.Size = sizeof(STOR_UNIT_DFX_SETTINGS);
    
    // 3. ���� F-state ����������
    DfxSettings.UnitDfxSettings.FStateCount = 2; // F0 �� F1
    DfxSettings.UnitDfxSettings.DeepestWakeableFState = 0;
    
    // 4. ���� F-states
    DfxSettings.UnitDfxSettings.FStates[0].Version = 
        STOR_UNIT_DFX_F_STATE_VERSION_V1;
    DfxSettings.UnitDfxSettings.FStates[0].Size = 
        sizeof(STOR_UNIT_DFX_F_STATE);
    DfxSettings.UnitDfxSettings.FStates[0].TransitionLatency = 0;
    DfxSettings.UnitDfxSettings.FStates[0].ResidencyRequirement = 0;
    DfxSettings.UnitDfxSettings.FStates[0].NominalPower = 
        STOR_POFX_UNKNOWN_POWER;

    DfxSettings.UnitDfxSettings.FStates[1].Version = 
        STOR_UNIT_DFX_F_STATE_VERSION_V1;
    DfxSettings.UnitDfxSettings.FStates[1].Size = 
        sizeof(STOR_UNIT_DFX_F_STATE);
    DfxSettings.UnitDfxSettings.FStates[1].TransitionLatency = 1000; // 1ms
    DfxSettings.UnitDfxSettings.FStates[1].ResidencyRequirement = 2000; // 2ms
    DfxSettings.UnitDfxSettings.FStates[1].NominalPower = 
        STOR_POFX_UNKNOWN_POWER;

    // 5. ��ʼ�� DFx
    status = StorPortInitializeDfx(DeviceExtension, &DfxSettings);
    if (!NT_SUCCESS(status)) {
        return SP_RETURN_ERROR;
    }

    return SP_RETURN_FOUND;
}
```

2. ʵ�� Unit ����� DFx �ص�:

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
SCSI_ADAPTER_CONTROL_STATUS 
HwAdapterControl(
    _In_ PVOID DeviceExtension,
    _In_ SCSI_ADAPTER_CONTROL_TYPE ControlType,
    _In_ PVOID Parameters
)
{
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;
    
    switch(ControlType) {
        case ScsiUnitPoFxPowerRequired: {
            PSCSI_UNIT_POFX_POWER_REQUIRED Params = 
                (PSCSI_UNIT_POFX_POWER_REQUIRED)Parameters;
            
            // ���� Unit �ĵ�Դ�ָ�����
            if (!card_power_on(&pdx->card)) {
                return ScsiAdapterControlFailure;
            }
            
            // �ȴ����ȶ�
            StorPortStallExecution(pdx->cfg->card_power_stable_time_us);
            break;
        }
        
        case ScsiUnitPoFxPowerActive: {
            PSCSI_UNIT_POFX_POWER_ACTIVE Params = 
                (PSCSI_UNIT_POFX_POWER_ACTIVE)Parameters;
            
            // ��ɿ��ĳ�ʼ��
            if (!card_init(&pdx->card)) {
                return ScsiAdapterControlFailure;
            }
            break;
        }
        
        case ScsiUnitPoFxPowerSetFState: {
            PSCSI_UNIT_POFX_SET_F_STATE Params = 
                (PSCSI_UNIT_POFX_SET_F_STATE)Parameters;
            
            // ���� F-state �л�
            switch(Params->FState) {
                case 0: // F0
                    if (!card_set_active_state(&pdx->card)) {
                        return ScsiAdapterControlFailure;
                    }
                    break;
                    
                case 1: // F1
                    if (!card_set_low_power_state(&pdx->card)) {
                        return ScsiAdapterControlFailure;
                    }
                    break;
            }
            break;
        }
    }
    
    return ScsiAdapterControlSuccess;
}
```

3. �� I/O ·����ʹ�� DFx API:

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
BOOLEAN 
HwStartIo(
    _In_ PVOID DeviceExtension,
    _In_ PSCSI_REQUEST_BLOCK Srb
)
{
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;
    ULONG status;
    
    // 1. ��ȡ Unit �� Active ����
    status = StorPortUnitPoFxActivateComponent(
        DeviceExtension,
        Srb->PathId,
        Srb->TargetId,
        Srb->Lun,
        NULL,
        NULL,
        0,
        0
    );
    
    if (status == STOR_STATUS_BUSY) {
        // Unit ���ڻָ����ӳٴ���
        QueueDeferredSrb(pdx, Srb);
        return TRUE;
    } else if (status != STOR_STATUS_SUCCESS) {
        StorPortCompleteRequest(DeviceExtension, 
                              Srb->PathId,
                              Srb->TargetId,
                              Srb->Lun,
                              SRB_STATUS_ERROR);
        return TRUE;
    }
    
    // 2. ���� I/O
    BOOLEAN result = ProcessSrbRequest(pdx, Srb);
    
    // 3. �ͷ� Unit ����
    StorPortUnitPoFxIdleComponent(
        DeviceExtension,
        Srb->PathId,
        Srb->TargetId,
        Srb->Lun,
        NULL,
        NULL,
        0,
        0
    );
    
    return result;
}
```

4. ʵ�� idle timeout ֧��:

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
// �� HwFindAdapter ������
pdx->os.UnitPoFxDevice.Flags |= STOR_POFX_DEVICE_FLAG_IDLE_TIMEOUT;
pdx->os.UnitPoFxDevice.UnitIdleTimeoutInMS = 
    pdx->cfg->feature_item.card_idle_timeout_ms;
```

�ؼ���˵��:

1. DFx ��ʼ��:
- ���� F-states ��������
- ������ȷ���ӳٺ�פ��ʱ��
- ���� idle timeout ����

2. Unit ����ص�:
- PowerRequired: ������Դ�ָ�
- PowerActive: ������ʼ��
- SetFState: F-state �л�

3. ������:
- ���лص�����Ҫ������ȷ��״̬
- ʵ�ֳ�ʱ�����Ի���
- ��ȷ������Դ

4. pwrtest ֧��:
- ȷ����ȷ��Ӧ F-state �л�
- ʵ�ֿɿ��� idle timeout
- ֧�ֵ�Դ״̬ͳ��

������ʵ�ֿ���:
1. ֧�� SD ����ϸ���ȵ�Դ����
2. ͨ�� pwrtest ��֤
3. �ṩ�ɿ��ĵ�Դ������

## User:


[text]  
�ղŵĽ��鲻����Storport�ĵ��������� @Web @https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-_stor_pofx_device_v3 Remarks
To register a storage adapter for Storport PoFx support, the miniport driver calls StorPortEnablePassiveInitialization in its HwStorInitialize routine and implements a HwStorPassiveInitializeRoutine. The miniport calls StorPortInitializePoFxPower within it's HwStorPassiveInitializeRoutine to provide information about the adapter component.

To register a storage unit for Storport PoFx support, the miniport driver implements the HwStorUnitControl callback routine and provides handling of the ScsiUnitPoFxPowerInfo unit control code. When the handling the ScsiUnitPoFxPowerInfo control code, the miniport calls StorPortInitializePoFxPower if idle power management for the unit component is enabled.

The component for the storage device identified by its Components array index. Storage devices have only one component so the index of 0 is used. Routines such as StorPortPoFxActivateComponent and StorPortPoFxIdleComponent use the array index of a component to identify the component.


## AI (claude-3.5-sonnet):



��˵�öԣ��Ҹ��� Storport �ĵ���������ʵ�ַ�����

1. ������ HwStorInitialize �����ñ�����ʼ����

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
BOOLEAN 
HwStorInitialize(
    _In_ PVOID DeviceExtension
)
{
    // ���ñ�����ʼ��
    return StorPortEnablePassiveInitialization(DeviceExtension);
}
```

2. �� HwStorPassiveInitializeRoutine �г�ʼ���������� PoFx��

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
BOOLEAN 
HwStorPassiveInitializeRoutine(
    _In_ PVOID DeviceExtension
)
{
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;
    BOOLEAN enableD3Cold = FALSE;
    ULONG status;

    // ��ʼ�������� PoFx
    STOR_POFX_DEVICE AdapterPoFxDevice = {0};
    AdapterPoFxDevice.Version = STOR_POFX_DEVICE_VERSION_V3;
    AdapterPoFxDevice.Size = sizeof(STOR_POFX_DEVICE);
    AdapterPoFxDevice.ComponentCount = 1;  // �洢�豸ֻ��һ�����
    
    // ������� 0
    AdapterPoFxDevice.Components[0].Version = STOR_POFX_COMPONENT_VERSION_V1;
    AdapterPoFxDevice.Components[0].Size = sizeof(STOR_POFX_COMPONENT);
    AdapterPoFxDevice.Components[0].FStateCount = 2;
    AdapterPoFxDevice.Components[0].DeepestWakeableFState = 0;
    
    // ���� idle timeout
    AdapterPoFxDevice.Flags |= STOR_POFX_DEVICE_FLAG_IDLE_TIMEOUT;
    AdapterPoFxDevice.AdapterIdleTimeoutInMS = 
        pdx->cfg->feature_item.psd_mode.adapter_idle_time_s * 1000;

    status = StorPortInitializePoFxPower(
        DeviceExtension,
        NULL,
        &AdapterPoFxDevice,
        &enableD3Cold
    );

    return (status == STOR_STATUS_SUCCESS);
}
```

3. ʵ�� HwStorUnitControl ���� Unit ����� PoFx��

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
BOOLEAN 
HwStorUnitControl(
    _In_ PVOID DeviceExtension,
    _In_ SCSI_UNIT_CONTROL_TYPE ControlType,
    _In_ PVOID Parameters
)
{
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;
    
    switch(ControlType) {
        case ScsiUnitPoFxPowerInfo: {
            PSCSI_UNIT_POFX_POWER_INFO PowerInfo = 
                (PSCSI_UNIT_POFX_POWER_INFO)Parameters;
            BOOLEAN enableD3Cold = FALSE;
            
            // ����Ƿ���ҪΪ�� Unit ���� idle power management
            if (IsUnitIdlePowerManagementEnabled(pdx, PowerInfo)) {
                STOR_POFX_DEVICE UnitPoFxDevice = {0};
                UnitPoFxDevice.Version = STOR_POFX_DEVICE_VERSION_V3;
                UnitPoFxDevice.Size = sizeof(STOR_POFX_DEVICE);
                UnitPoFxDevice.ComponentCount = 1;
                
                // ���� Unit ���
                UnitPoFxDevice.Components[0].Version = STOR_POFX_COMPONENT_VERSION_V1;
                UnitPoFxDevice.Components[0].Size = sizeof(STOR_POFX_COMPONENT);
                UnitPoFxDevice.Components[0].FStateCount = 2;
                UnitPoFxDevice.Components[0].DeepestWakeableFState = 0;
                
                // ���� Unit idle timeout
                UnitPoFxDevice.Flags |= STOR_POFX_DEVICE_FLAG_IDLE_TIMEOUT;
                UnitPoFxDevice.UnitIdleTimeoutInMS = 
                    pdx->cfg->feature_item.card_idle_timeout_ms;

                ULONG status = StorPortInitializePoFxPower(
                    DeviceExtension,
                    PowerInfo->UnitParameters,  // Unit ����
                    &UnitPoFxDevice,
                    &enableD3Cold
                );
                
                return (status == STOR_STATUS_SUCCESS);
            }
            return TRUE;
        }
    }
    
    return FALSE;
}
```

4. ����Ҫ�����豸ʱʹ�� PoFx API��

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
BOOLEAN 
HwStartIo(
    _In_ PVOID DeviceExtension,
    _In_ PSCSI_REQUEST_BLOCK Srb
)
{
    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;
    ULONG status;
    
    // ��ȡ����������� Active ���� (������� 0)
    status = StorPortPoFxActivateComponent(
        DeviceExtension,
        NULL,
        NULL,
        0,  // ������� 0
        0
    );
    
    if (status == STOR_STATUS_BUSY) {
        // �豸���ڻָ�
        return FALSE;
    }
    
    // ���� I/O
    BOOLEAN result = ProcessSrbRequest(pdx, Srb);
    
    // �ͷ��������������
    StorPortPoFxIdleComponent(
        DeviceExtension,
        NULL,
        NULL,
        0,  // ������� 0
        0
    );
    
    return result;
}
```

�ؼ�������

1. ��ʼ�����̣�
- ʹ�� HwStorInitialize �� HwStorPassiveInitializeRoutine
- ����ȷ�Ļص��г�ʼ�� PoFx

2. Unit ֧�֣�
- ͨ�� HwStorUnitControl �� ScsiUnitPoFxPowerInfo ʵ��
- Ϊÿ�� Unit �������� PoFx

3. ���������
- �洢�豸ֻʹ��һ����������� 0��
- ���� PoFx API ����ʹ��������� 0

������ʵ�ָ����� Storport �淶Ҫ��

## User:


[selections]  
```c
HW_PASSIVE_INITIALIZE_ROUTINE	StorHwPassiveInitialize;

```
```c
scsi_HwUnitControl

```
[text]  
���鵱ǰ���� @Codebase   @StorHwPassiveInitialize  @@scsi_HwUnitControl(PVOID, SCSI_UNIT_CONTROL_TYPE, PVOID) ����Ϊ�Ѿ����Ϲ淶


## AI (claude-3.5-sonnet):



�����������Ϊ��ǰʵ�ֻ������� Storport �淶�����м�����Ҫע������ƣ�

1. StorHwPassiveInitialize ʵ�ֻ�����ȷ��

```366:448:SDSTORDriver/sdstor/windows_os/winscsientry.c
BOOLEAN
StorHwPassiveInitialize(IN PVOID DeviceExtension)
{
    DbgInfo(MODULE_OS_ENTRY, FEATURE_PM_TRACE, NOT_TO_RAM, "Enter %s\n", __FUNCTION__);

    UCHAR i;
    ULONG status = STOR_STATUS_SUCCESS;
    BOOLEAN enableD3Cold = FALSE;

    bht_dev_ext_t *pdx = (bht_dev_ext_t *)DeviceExtension;

    req_global_init(pdx); //todo failed then FindAdapter failed
    DevMapAdd(pdx); // call this to add pdx to Map   

    // Registry RTD3 en controls POfx register process.
    if (pdx->pm_state.rtd3_en != TRUE)
        goto Exit;

    //
    // Get and cache D3Cold support.  
    //
     status = StorPortGetD3ColdSupport(DeviceExtension, NULL, &enableD3Cold);

     if (status == STOR_STATUS_SUCCESS) {
         DbgInfo(MODULE_OS_ENTRY, FEATURE_PM_TRACE, NOT_TO_RAM, "d3Coldsupported %0x\n", enableD3Cold);
     }

    os_memset((PCHAR)(&(pdx->os.AdptPoFxDevice)), 0, sizeof(STOR_POFX_DEVICE_V2));

    pdx->os.AdptPoFxDevice.Version = STOR_POFX_DEVICE_VERSION_V2;
    pdx->os.AdptPoFxDevice.Size = STOR_POFX_DEVICE_V2_SIZE;
    pdx->os.AdptPoFxDevice.ComponentCount = 1;


    // indicate dump miniport can't bring adapter to active
    pdx->os.AdptPoFxDevice.Flags |= STOR_POFX_DEVICE_FLAG_NO_DUMP_ACTIVE;

    pdx->os.AdptPoFxDevice.Components[0].Version = STOR_POFX_COMPONENT_VERSION_V1;
    pdx->os.AdptPoFxDevice.Components[0].Size = STOR_POFX_COMPONENT_SIZE;
    pdx->os.AdptPoFxDevice.Components[0].FStateCount = 1;
    pdx->os.AdptPoFxDevice.Components[0].DeepestWakeableFState = 0;
    pdx->os.AdptPoFxDevice.Components[0].Id = STORPORT_POFX_ADAPTER_GUID;

    pdx->os.AdptPoFxDevice.Components[0].FStates[0].Version = STOR_POFX_COMPONENT_IDLE_STATE_VERSION_V1;
    pdx->os.AdptPoFxDevice.Components[0].FStates[0].Size = STOR_POFX_COMPONENT_IDLE_STATE_SIZE;
    pdx->os.AdptPoFxDevice.Components[0].FStates[0].TransitionLatency = 0;
    pdx->os.AdptPoFxDevice.Components[0].FStates[0].ResidencyRequirement = 0;
    pdx->os.AdptPoFxDevice.Components[0].FStates[0].NominalPower = STOR_POFX_UNKNOWN_POWER;

    // registry runtime power management for Adapter
    pdx->os.AdptPoFxDevice.Flags |= STOR_POFX_DEVICE_FLAG_IDLE_TIMEOUT;
    pdx->os.AdptPoFxDevice.Flags |= STOR_POFX_DEVICE_FLAG_ADAPTER_D3_WAKE;

    //pdx->os.AdptPoFxDevice.AdapterIdleTimeoutInMS = 1000; //10ms Idle tiemout for D3.
    pdx->os.AdptPoFxDevice.AdapterIdleTimeoutInMS = pdx->cfg->feature_item.psd_mode.adapter_idle_time_s * 1000; //The unit is 1ms

    status = StorPortInitializePoFxPower(DeviceExtension, NULL, (PSTOR_POFX_DEVICE)(&(pdx->os.AdptPoFxDevice)), &enableD3Cold);
    DbgInfo(MODULE_OS_ENTRY, FEATURE_PM_TRACE, NOT_TO_RAM, "%s StorPortInitializePoFxPower=%x,enableD3Cold=%x\n",
            __TIME__, status, enableD3Cold);
    DbgInfo(MODULE_OS_ENTRY, FEATURE_PM_TRACE, NOT_TO_RAM, "pdx->os.AdptPoFxDevice.AdapterIdleTimeoutInMS =%d ,pdx->os.AdptPoFxDevice.Flags=%x , STOR_STATUS_SUCCESS=%x\n",
            pdx->os.AdptPoFxDevice.AdapterIdleTimeoutInMS, pdx->os.AdptPoFxDevice.Flags, STOR_STATUS_SUCCESS);

    if(status != STOR_STATUS_SUCCESS)
    {
        pdx->os.StateFlags.PoFxEnabled = FALSE;
        pdx->os.StateFlags.PoFxActive = FALSE;

        goto Exit;
    }

    // register success
    // win_rtd3_req_wait_wake(pdx);

    pdx->os.StateFlags.PoFxEnabled = TRUE;
    pdx->os.StateFlags.PoFxActive = TRUE;

Exit:
    //create work thread at end of stor-initialization entry.
    os_create_thread(pdx, &pdx->os.thread[0], pdx, thread_main);

    DbgInfo(MODULE_OS_ENTRY, FEATURE_PM_TRACE, NOT_TO_RAM, "Exit %s\n", __FUNCTION__);
    return TRUE;
}
```


��Ҫ������
- ��ȷ��ʼ�� STOR_POFX_DEVICE_V2 �ṹ
- ��������ȷ�� Version �� ComponentCount
- ������ IDLE_TIMEOUT ��־
- ���� StorPortInitializePoFxPower

2. ���� HwAdapterControl �е� PoFx ����������

```1977:1979:SDSTORDriver/sdstor/windows_os/winscsientry.c
    case ScsiAdapterPoFxPowerControl:

        break;
```


��Ҫ��� ScsiAdapterPoFxPowerControl �Ĵ���

```c
case ScsiAdapterPoFxPowerControl:
{
    PSCSI_PO_FX_POWER_CONTROL poFxControl = 
        (PSCSI_PO_FX_POWER_CONTROL)Parameters;
        
    switch(poFxControl->PowerControlCode) {
        case ScsiPoFxPowerControlComponentActive:
            // ����豸��ʼ��
            if (pdx->pm_state.d0d3_state == StorPowerDeviceD0) {
                pcr_part_a_restore(pdx);
                pcr_part_b_restore(pdx);
            }
            break;
            
        case ScsiPoFxPowerControlComponentIdle:
            // ׼������͹���״̬
            pcr_part_a_backup(pdx);
            pcr_part_b_backup(pdx);
            break;
    }
    break;
}
```

3. Unit ����� PoFx ֧���Ѿ�ʵ�֣�

```506:530:SDSTORDriver/sdstor/windows_os/winscsientryapi.c
    if(sd_bus_PoFxEnabled(sdbusport_ext))
    {
        ULONG status;

        status = StorPortPoFxActivateComponent((bht_dev_ext_t *)(sdbusport_ext->AdapterExtension),
                                               (PSTOR_ADDRESS)&sdbusport_ext->DeviceExtension[0].DeviceAddress,
                                               (PSCSI_REQUEST_BLOCK)Srb,
                                               0,
                                               0);
        // STOR_STATUS_BUSY indicates that ActivateComponent is not completed yet.
        idle = (status == STOR_STATUS_BUSY);
        DbgInfo(MODULE_OS_ENTRYAPI, FEATURE_PM_TRACE, NOT_TO_RAM, "rtget---StorPortPoFxActivateComponent =%x\n", status);

        if((status == STOR_STATUS_SUCCESS) || (status == STOR_STATUS_BUSY))
            referenceAcquired = TRUE;

    }
    else
        UNREFERENCED_PARAMETER(Srb);

    if(InIdle != NULL)
        *InIdle = idle;

    return referenceAcquired;
}
```


������
- ��ȷʹ�� StorPortPoFxActivateComponent
- ���� STOR_STATUS_BUSY ״̬
- ʵ�������ü�������

����Ľ���

1. �� StorHwPassiveInitialize ����Ӵ�����
```c
if (status != STOR_STATUS_SUCCESS) {
    DbgErr("StorPortInitializePoFxPower failed: %x\n", status);
    return FALSE;  // ���� FALSE ��ʾ��ʼ��ʧ��
}
```

2. ���ƼĴ������ʵ�״̬��飺

```295:306:SDSTORDriver/sdstor/main/pmfunc.c
        if (pdx->pm_state.d0d3_state == StorPowerDeviceD0) {
            pci_writel(&pdx->host, 0xdc, pdx->pm_state.reg_0xdc);
            pci_writel(&pdx->host, 0x3e0, pdx->pm_state.reg_0x3e0);
            pci_writel(&pdx->host, 0x3e8, pdx->pm_state.reg_0x3e8);
            pci_writel(&pdx->host, 0x3ec, pdx->pm_state.reg_0x3ec);
            pci_writel(&pdx->host, 0xf4, pdx->pm_state.reg_0xf4);
            pci_writel(&pdx->host, 0x74, pdx->pm_state.reg_0x74);
            pci_writel(&pdx->host, 0xf0, pdx->pm_state.reg_0xf0);
            pci_writel(&pdx->host, 0x90, pdx->pm_state.reg_0x90);
        }
        else { //D3
            pci_cfgio_writel(&pdx->host, 0xdc, pdx->pm_state.reg_0xdc);
```


�ⲿ�ִ����Ѿ���ȷ������ D0/D3 ״̬�µļĴ������ʲ��졣

������˵����ǰʵ�ַ��� Storport �淶��ֻ��Ҫ���� ScsiAdapterPoFxPowerControl �Ĵ����ɡ�

## User:


[text]  
ScsiAdapterPoFxPowerControl�Ĺ�����ʲô���������, ��ο� @Codebase @Web 


## AI (claude-3.5-sonnet):

���� [Storport PoFx �ĵ�](https://learn.microsoft.com/en-us/windows-hardware/drivers/storage/storport-pofx) �ʹ���⣬ScsiAdapterPoFxPowerControl �� PoFx �������֪ͨ miniport �������״̬�仯�Ļص���

��Ҫ�� scsi_HwAdapterControl �����ƴ˴���

```c:SDSTORDriver/sdstor/windows_os/winscsientry.c
case ScsiAdapterPoFxPowerControl:
{
    PSCSI_PO_FX_POWER_CONTROL poFxControl = 
        (PSCSI_PO_FX_POWER_CONTROL)Parameters;
    
    switch(poFxControl->PowerControlCode) {
        case ScsiPoFxPowerControlComponentIdle:
            // ����������״̬
            if (pdx->pm_state.d0d3_state == StorPowerDeviceD0) {
                // ����Ĵ���״̬���ο����д���:
                // SDSTORDriver/sdstor/main/pmfunc.c startLine: 290 endLine: 306
                req_pre_enter_d3(pdx);
            }
            break;
            
        case ScsiPoFxPowerControlComponentActive:
            // ����ָ��״̬
            if (pdx->pm_state.d0d3_state == StorPowerDeviceD0) {
                // �ָ��豸״̬
                req_exit_d3(pdx);
                
                // �����ӳٵ�����
                ProcessDeferredRequests(pdx);
            }
            break;
            
        case ScsiPoFxPowerControlComponentIdleState:
            // F-state �ı�֪ͨ
            ULONG newFState = poFxControl->PowerParameters.FState;
            if (newFState == 0) {
                // F0 ״̬ - ��ȫ����
                pdx->pm_state.current_fstate = 0;
            } else {
                // F1 �����ĵ͹���״̬
                pdx->pm_state.current_fstate = newFState;
            }
            break;
    }
    break;
}
```

�ؼ��㣺

1. �˻ص��� StorPortPoFxActivateComponent/IdleComponent ��ϣ�

```506:530:SDSTORDriver/sdstor/windows_os/winscsientryapi.c
    if(sd_bus_PoFxEnabled(sdbusport_ext))
    {
        ULONG status;

        status = StorPortPoFxActivateComponent((bht_dev_ext_t *)(sdbusport_ext->AdapterExtension),
                                               (PSTOR_ADDRESS)&sdbusport_ext->DeviceExtension[0].DeviceAddress,
                                               (PSCSI_REQUEST_BLOCK)Srb,
                                               0,
                                               0);
        // STOR_STATUS_BUSY indicates that ActivateComponent is not completed yet.
        idle = (status == STOR_STATUS_BUSY);
        DbgInfo(MODULE_OS_ENTRYAPI, FEATURE_PM_TRACE, NOT_TO_RAM, "rtget---StorPortPoFxActivateComponent =%x\n", status);

        if((status == STOR_STATUS_SUCCESS) || (status == STOR_STATUS_BUSY))
            referenceAcquired = TRUE;

    }
    else
        UNREFERENCED_PARAMETER(Srb);

    if(InIdle != NULL)
        *InIdle = idle;

    return referenceAcquired;
}
```


2. ��Ҫ�� D-state ״̬ͬ����

```1870:1888:SDSTORDriver/sdstor/windows_os/winscsientry.c
        DbgInfo(MODULE_OS_ENTRY, FEATURE_PM_TRACE, NOT_TO_RAM, "ScsiAdapterPower - %s\n",
                adpt_pow->PowerState == StorPowerDeviceD0 ? "D0" : "D3");
        PrintMsg("ScsiAdapterPower - %s!\n", adpt_pow->PowerState == StorPowerDeviceD0 ? "D0" : "D3");
        
        //Record the D-state, WDM driver must handle it well. Every time D-state is changed, it must be changed here.
        pdx->pm_state.d0d3_state = adpt_pow->PowerState;

        if(StorPowerActionNone == adpt_pow->PowerAction
                && StorPowerDeviceD3  == adpt_pow->PowerState)
        {
            DbgInfo(MODULE_OS_ENTRY, FEATURE_PM_TRACE, NOT_TO_RAM, "Enter Runtime D3!\n");
            PrintMsg("Enter Runtime D3!\n");
            pdx->pm_state.rtd3_entered = TRUE;
            pdx->pre_card.s3s4_resume_for_card_init = TRUE;
            // if (FALSE == os_pending_thread(pdx, TRUE))  //for thread sync : such as autotimer stop infinite
            //     DbgErr("%s pending thread failed\n", __FUNCTION__);
            req_pre_enter_d3(pdx);
            req_enter_d3(pdx);
            //os_pending_thread(pdx, FALSE);
```


3. ��Ҫ���ܣ�
- �����������/�˳�����״̬
- ���� F-state �仯
- ȷ���� D-state ״̬��ȷЭ��
- �����豸�Ĵ���״̬

4. ע�����
- ֻ�� D0 ״̬�·����豸�Ĵ���
- ��ȷ�����ӳٵ�����
- ά����ȷ�Ĺ���״̬��־

������ʵ�ֿ���ȷ�� PoFx �������豸״̬�������ȷЭ����
