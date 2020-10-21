
# emr efcore review

## 必须解决

### **客户端求值**

+   如where条件包含的GetValueOrDefault()不能被翻译成sql语句
+   不规范代码段例子

```
     public async Task<List<SysRole>> GetRolesAsync()
            {
                var results = _context.SysRole
                    .Where(_p => _p.State.GetValueOrDefault() == 1)
                    .ToList()
                    .OrderBy(_p => (_p.RoleName + "").GetSpellCode())
                    .ToList();

                return results;
            }
```
```
    public async Task<AnnoucemenDataOutput> AnnoucementData(int id)
            {
                var data = await _dbContext.SysAnnouncement.Where(q => !q.IsDeleted.GetValueOrDefault() && q.State == true)
                    .Where(q => q.Id == id).Select(q => new AnnoucemenDataOutput
                    {
                        Id = q.Id,
                        ValidDate = q.ValidDate.GetValueOrDefault().DateTimeOffset2TimeStamp(),
                        DeptCodes = q.DeptCode,
                        PublishType = q.PublishType,
                        PublishTime = q.PublishTime,
                        Data = q.Data,
                        Title = q.Title,
                        Author = q.Author
                    }).FirstOrDefaultAsync();
                return data;
            }
```

### **try catch 滥用**

+   没必要使用的地方使用
+   吃掉了真实的异常信息
+   不规范代码段例子

```
            public async Task<bool> SaveFolder(EmrMedicalRecordFolder recordFolder)
            {
                try
                {
                    _dbContext.EmrMedicalRecordFolder.Update(recordFolder);
                    await _dbContext.SaveChangesAsync();
                    return true;
                }
                catch (Exception)
                {
                    throw new InternalServerErrorException("文件夹目录修改失败");
                }
            }
```
```
    public async Task<int> UpdateRecord(CpoeInpatShiftDept arg)
            {
                try
                {
                    _context.Entry<CpoeInpatShiftDept>(arg).State = EntityState.Modified;
                    var result = await _context.SaveChangesAsync();
                    return result;
                }
                catch
                {
                    return 0;
                }
            }
```
```
     public async Task<int> SaveAntibacterialAsync(AntibacterialApplyInput input)
            {
                try
                {
                    var result = 0;
                    var apply = Mapper.Map<CpoeAntibacterialApply>(input);
                    apply.InputTime = DateTimeOffset.UtcNow;
                    _dbContext.CpoeAntibacterialApply.Add(apply);
                    result = await _dbContext.SaveChangesAsync();
       
                    return result;
                }
                catch (Exception ex)
                {
                    return 0;
                }  
            }
```
```
            public async Task<int> SaveAntibacterialListAsync(List<AntibacterialApplyInput> input)
            {
                try
                {
                    var result = 0;
                    var applyList = Mapper.Map<List<CpoeAntibacterialApply>>(input);
                    applyList.ForEach(o=> {
                        o.InputTime = DateTimeOffset.UtcNow;
                    });
                    await  _dbContext.CpoeAntibacterialApply.AddRangeAsync(applyList);
                    result = await _dbContext.SaveChangesAsync();

                    return result;
                }
                catch (Exception ex)
                {
                    return 0;
                }
            }
```
```
    public async Task<HerbOrderGetOutput> GetHerbOrderListByOrderIdAsync(int orderId)
            {
                var result = new HerbOrderGetOutput();
                try
                {
                    var herbOrder =await _dbContext.CpoeHerbOrder.Where(o => o.OrderId == orderId).FirstOrDefaultAsync();
                    result.Order = Mapper.Map<HerbOrderOutput>(herbOrder);
                    var herbOrderDetails = await _dbContext.CpoeHerbOrderDetail.Where(o => o.HerbOrderId == herbOrder.HerbOrderId).ToListAsync();
                    result.Detail = Mapper.Map<List<HerbDetailOutput>>(herbOrderDetails);
                    return result;

                }
                catch(Exception ex)
                {
                    return result;
                }
            }
```
```
    public async Task SaveHerbOrdersAsync(List<SaveHerbOrderInput> herbOrderInputs)
            {
                try
                {
                    if (herbOrderInputs == null || !Enumerable.Any(herbOrderInputs))
                    {
                        return;
                    }
                    if (herbOrderInputs != null || Enumerable.Any(herbOrderInputs))
                    {
                        herbOrderInputs.ForEach(async (o) => {
                            var herbOrder = Mapper.Map<CpoeHerbOrder>(o.Order);
                            if (herbOrder != null)
                            {
                                herbOrder.InputTime = DateTimeOffset.Now;
                                var herbOrderDetails = Mapper.Map<List<CpoeHerbOrderDetail>>(o.Detail);
                                await _dbContext.CpoeHerbOrder.AddRangeAsync(herbOrder);
                                await _dbContext.CpoeHerbOrderDetail.AddRangeAsync(herbOrderDetails);
                            }
                        });
                        await _dbContext.SaveChangesAsync();
                    }

                }
                catch (Exception ex)
                {

                }
            }
```

## 建议解决

### **无用追踪**

+   无须追踪的数据没有加AsNoTracking
+   不规范代码段例子

```
     public async Task<bool> CheckInpatDiagByDiagType(long inpaId, EnumDiagType diagType)
            {
                var diagTy = ((int)diagType).ToString();
                var check = await _dbContext.CpoeDiagnoseRecord.Where(m =>
                    m.InpatId == inpaId && m.IsDeleted == false && m.DiagType == diagTy).ToListAsync();

                return check.Count > 0;
            }
```

### **没有使用异步方法**

+   没有优先使用异步方法
+   不规范代码段例子

```
     public async Task<int> AddDiagnoseBatch(IEnumerable<PubDiagnose> diagnoses, int emplid)
            {
                if (diagnoses == null)
                {
                    throw new InvalidParameterException("参数不能为null");
                }

                if (!diagnoses.Any())
                {
                    return 0;
                }

                this._context.PubDiagnose.AddRange(diagnoses.Where(_p => _p != null));
                await this._context.SaveChangesAsync();
                return 1;

                //this._context.Diagnose.AddRange(diagnoses.Where(_p => _p != null));
                //return await this._context.SaveChangesAsync();
            }
```

### **事务滥用**

-   没必要使用事务的场景使用事务
-   不规范代码段例子

```
            public async Task<bool> UpdateQualityControl(List<CpoeRecordQualityControl> recordQuality, List<CpoeRecordQualityControlRule> controlRules)
            {
                using (var transaction = _dbContext.Database.BeginTransaction(IsolationLevel.ReadCommitted))
                {
                    try
                    {
                        _dbContext.CpoeRecordQualityControl.UpdateRange(recordQuality);
                        await _dbContext.SaveChangesAsync();

                        if (controlRules.Count > 0)
                        {
                            _dbContext.CpoeRecordQualityControlRule.UpdateRange(controlRules);
                            await _dbContext.SaveChangesAsync();
                        }

                        transaction.Commit();
                        return true;
                    }
                    catch (Exception ex)
                    {
                        transaction.Rollback();
                        throw new InternalServerErrorException($"状态改变失败，ErrorMessage:{ex.Message}\r\nInnerException:{ex.InnerException}", ex);
                    }
                }
            }
```

## 根据业务需求考虑是否解决

### **递归调用**

+   递归方法里面操作数据库
+   不规范代码段例子

```
    public async Task<List<PubDepartment>> GetParentDepts(List<string> subDeptsCode)
            {
                var result = new List<PubDepartment>();
                var depts = await _context.PubDepartment.Where(q => q.State == 1).Where(q => subDeptsCode.Contains(q.DeptCode)).AsNoTracking().ToListAsync();
                var temp = new List<string>();
                foreach (var item in depts)
                {
                    if (result.All(q => q.DeptCode != item.DeptCode))
                    {
                        result.Add(item);
                    }
                    if (!string.IsNullOrEmpty(item.ParentDeptCode))
                    {
                        temp.Add(item.ParentDeptCode);
                    }
                }
                if (temp.Any())
                {
                    result.AddRange(await GetParentDepts(temp));
                }

                return result;
            }
```
```
            public async Task GetSubDepts(int deptId, List<int> result)
            {
                var depts = await _context.PubDepartment
                    .Where(q => q.State == CommonData.ValidState)
                    .Where(q => q.ParentDeptId == deptId).AsNoTracking().ToListAsync();
                if (!depts.Any())
                {
                    result.Add(deptId);
                }
                depts.ForEach(async q => await GetSubDepts(q.DeptId, result));
            }
```

## 规范参考
```
    数据追踪参考规范： https://docs.microsoft.com/zh-cn/ef/core/querying/tracking

    客户端求值参考规范：https://docs.microsoft.com/zh-cn/ef/core/querying/client-eval

    异步查询参考规范：https://docs.microsoft.com/zh-cn/ef/core/querying/async

    加载相关数据参考规范：https://docs.microsoft.com/zh-cn/ef/core/querying/related-data

    事务使用参考规范：https://docs.microsoft.com/zh-cn/ef/core/saving/transactions
```