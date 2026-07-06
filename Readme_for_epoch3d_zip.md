EPOCH3D Drude-Lorentz Si 修改总结
根据 如何将 EPOCH 中自由电子的电磁响应改为电介质的响应?--Drude-Lorentz model.pdf，本次修改的目标不是直接修改激光边界注入的电场，而是在电子推动方程中修改电子受到的电场驱动力，使自由电子响应近似替换为硅介质的 Drude-Lorentz 响应。
默认物理假设为：只修改电子运动方程，离子不动或保持原有推动逻辑不变。电磁场仍通过 EPOCH 原有的电流沉积和 Maxwell 方程自洽演化。
实现方法
每个激光波长对应的 Drude-Lorentz 修正参数从 epoch3d/src/TABLE_Si/l_<lambda_nm> 读取，文件前三行依次为：
amp_drude_lorentz
sin_delta_phi
cos_delta_phi
电子推动时，先保留 EPOCH 插值得到的原始粒子位置电场 E_now，然后按
E_revised = |coe| * (E_now * cos(delta_phi)
          + E_cos_phase * sin(delta_phi))
进行修正。默认情况下，E_cos_phase 优先使用当前相位直接恢复：
E_cos_phase = E_now * cos(laser_phase) / sin(laser_phase)
这样可以避免粒子运动导致上一时刻电场与当前时刻电场幅值不一致时，把空间/包络变化混入相移计算。只有当 sin(laser_phase) 很小、直接相除可能导致数值发散时，才使用 PDF 中给出的上一时刻电场方法：
E_cos_phase = (E_now * cos(omega * dt) - E_last) / sin(omega * dt)
源码中保留了一个人为可选策略：
INTEGER, PARAMETER :: drude_lorentz_phase_method = c_dl_phase_direct_first
默认值 c_dl_phase_direct_first 表示优先直接相除；如需优先使用上一时刻缓存，可改为 c_dl_phase_cache_first。新粒子或缓存不可用时，会自动退回到可用的另一种方法。
修改文件
epoch3d/src/particles.F90
增加 USE utilities, ONLY: get_free_lun，用于 Fortran 2003 兼容的文件单元号获取。
保留并使用 epoch3d/src/TABLE_Si/l_<lambda_nm> 的读取逻辑，按第一束激光的 omega 计算整数纳米波长。
将 Drude-Lorentz 修正限制为：
species_list(ispecies)%species_type == c_species_id_electron
因此只修改电子推动方程，离子和其它非电子粒子不应用该修正。
在电子动量更新前，对 ex_part/ey_part/ez_part 应用修正后的电场。
新增 apply_drude_lorentz_to_e_part 的策略选择逻辑：默认优先通过 sin(laser_phase) 直接恢复幅值，仅在相位零点附近使用每个粒子缓存的 E_last 兜底；也可在源码常量中切换为缓存优先。
将表格读取中的 OPEN(newunit=...) 改为 get_free_lun() + OPEN(unit=...)，避免 Fortran 2003 构建报错。
所有相关改动均用注释 Drude-Lorentz Si modification 标注。
epoch3d/src/shared_data.F90
在 TYPE particle 中增加两个字段：
REAL(num), DIMENSION(3) :: drude_lorentz_e_part_last
REAL(num) :: drude_lorentz_e_part_last_valid
用于保存每个粒子上一时刻的原始插值电场，以及该缓存是否有效。
epoch3d/src/housekeeping/partlist.F90
将粒子打包变量数量 nvar 增加 4。
在 pack_particle 中保存上一时刻电场缓存。
在 unpack_particle 中恢复上一时刻电场缓存。
在 init_particle 中将缓存初始化为 0，并标记为无效。
这样粒子跨 MPI 边界迁移时，Drude-Lorentz 相移所需的上一时刻电场不会丢失。
epoch3d/src/TABLE_Si
将硅的波长响应表放到 epoch3d/src/TABLE_Si，使 EPOCH3D 源码相关的 Drude-Lorentz 参数与 epoch3d/src 保持在同一目录树下。
将生成这些表的脚本放到 epoch3d/src/generate_si_l_files.py，脚本会固定读写自身同目录下的 TABLE_Si，不再依赖从哪个目录启动。
程序按运行目录兼容查找：从 epoch3d/src 运行时，查找 TABLE_Si/l_<lambda_nm>。
从 epoch3d 运行时，查找 src/TABLE_Si/l_<lambda_nm>。
从工程根目录运行时，查找 epoch3d/src/TABLE_Si/l_<lambda_nm>。

验证结果
使用临时最小依赖模块，对以下改动相关文件完成了 Fortran 2003 语法级编译检查：shared_data.F90
partlist.F90
particles.F90

检查中发现并修复了 OPEN(newunit=...) 与项目 Fortran 2003 编译设置不兼容的问题。
完整 make -C epoch3d COMPILER=gfortran SDF=../SDF -j2 目前不能完成，因为当前工作区的 SDF 目录为空，缺少 SDF/Makefile、SDF/include/sdf.mod 和 SDF/lib/libsdf 等 EPOCH 构建依赖。这不是本次源码修改引入的问题。补齐真实 SDF 依赖后即可继续完整构建和运行测试。
运行注意事项
从工程根目录运行时，程序会查找 epoch3d/src/TABLE_Si/l_<lambda_nm>。
从 epoch3d 目录运行时，程序会查找 src/TABLE_Si/l_<lambda_nm>。
从 epoch3d/src 目录运行时，程序会查找 TABLE_Si/l_<lambda_nm>。
如果对应波长文件不存在，Drude-Lorentz 修正不会启用，程序会回到原始电子推动行为。
