###重复原因###
导致重复的原因是**`A5XS`**这个Seller是2014年注册的国家卖家，当初注册的时候相应的Warehouse表都是写入的USA的数据，在3期项目上的时候对线上老Seller进行了刷Warehouse数据，目前**`A5XS`**在`CodeCenter.dbo.Warehouse`表中USA已经被刷成了CHN，但是`CodeCenter.dbo.DropshipWarehouseMap`表中依然是USA。

这样就会导致Seller在Fulfillment Center中新加SBS的USA生成WarehouseNumber依然是CHN的，因为JOB会调用`SCM.[dbo].[UP_Seller_GenerateWarehouseNumber]`这个SP生成WarehouseNumber，而SP里面是用的`CodeCenter.dbo.DropshipWarehouseMap`表判断是否存在USA而非`CodeCenter.dbo.Warehouse`表。


由于`CodeCenter.dbo.DropshipWarehouseMap`表依然是USA，说以导致了**重复的WarehouseNumber！**

      SELECT TOP 1 @NextWarehouseNumber = RTRIM(WarehouseNumber) FROM CodeCenter.dbo.DropshipWarehouseMap WITH (NOLOCK)
      WHERE VendorNumber = @SellerID AND CountryCode = @CountryCode

      IF @NextWarehouseNumber IS NOT NULL
      BEGIN
        SET @WarehouseNumber = @NextWarehouseNumber
        RETURN
      END


###线上Seller###

目前线上有3个国际卖家存在此情况(*这3个Seller已经操作了Fulfillment Center*)

	use mktpls
	go

	select c1.SellerID,s.SellerID,s.Indate,c1.WarehouseCountryCode,c1.ShipByNewegg,c1.WarehouseNumber from  dbo.MPS_Seller_FulfillmentCenter as c1 with(nolock)
	left join dbo.EDI_Seller_BasicInfo as s with(nolock)
		on c1.SellerID=s.sellerid
	where c1.WarehouseNumber in 
	(
		select c.WarehouseNumber from dbo.MPS_Seller_FulfillmentCenter as c with(nolock)
		where c.ShipByNewegg=0
		group by c.WarehouseNumber
		having count(c.WarehouseNumber)>1
	)
	order by s.SellerID,c1.indate asc

图片

###其他的表###
目前WarehouseNumber相关表有如下：

1. *CodeCenter.dbo.Warehouse*
2. *CodeCenter.dbo.DropshipWarehouseMapAll*
3. *CodeCenter.dbo.DropshipWarehouseMap*
4. *CodeCenter.dbo.DropshipWarehouseMapHistory*
5. *CodeCenter.dbo.EDI_VendorWarehouseMap*

其他的Map表是否需要也刷新？

###解决方案###
目前只有3个Seller存在问题，但是如果其他之前注册的国际卖家操作Fulfillment Center添加SBS USA的数据也会出现这样的数据，所以应该先把刷数据的SQL添加对`CodeCenter.dbo.DropshipWarehouseMap`表的修改(需要注意区分是否是在3期项目之前注册的Seller)。

对于已经存在问题的3个Seller，较好的解决方案是将`CodeCenter.dbo.DropshipWarehouseMap`表中的USA修改为Seller注册时选择的Country，然后调用UP_Seller_GenerateWarehouseNumber进行生成WarehouseNumber，再更新Fulfillment Center。


> **详细细节需要周二早上电话商议。**

