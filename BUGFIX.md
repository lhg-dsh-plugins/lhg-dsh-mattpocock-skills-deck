# 本版修复（BUGFIX）

## 标题

挂「跨会话在办表」时原地改写按工作区共享的缓存快照对象：一个会话的表会被留在这个共享对象上，另一个会话经不重挂表的返回路径拿到同一份快照时，读到的就是上一个会话的表（上游 issue #1358）。本版改为挂表前先浅拷贝一份再写，共享对象不再被改。

版本：上游 1.9.0-beta.9 → 本版 1.9.0-beta.10。

## 证据链

1. **上游 issue #1358 的静态读码结论**（在本版固定的上游提交 a3014b1d 上逐处复核，全部属实）：
   - `src/host/snapshotEnvelope.ts` 的 `withWorkRouteTable` 把读数直接写进传进来的回包：`reply[field] = await d.workRouteTableReadout(...)`，然后原样返回同一个对象；
   - `src/host/sessionSnapshot.ts` 的 `shortCircuit()` 在选择没变时返回缓存里的同一个快照对象，随后 `handleSnapshot` 就在这个对象上挂表；
   - `src/host/index.ts` 的 `setCache` 把调用方给的对象引用直接放进按工作区分的缓存表，不做拷贝；
   - `src/host/commentThreads.ts` 的增量回执把缓存里的 `snapshot` 对象原样放进回包，这条路径不重挂表。
   表里每一行「是不是本会话」是按请求它的那个会话算的（`src/deck/shell/work-path-readout.ts` 的 `currentSession = (r.session === sessionId)`），所以留在共享对象上的表对下一个会话就是错的。
2. **最小复现**（离线，用假的读数出口构造信封，读数按请求会话返回标明归属的表）：让缓存里的快照对象先经 `withWorkRouteTable` 服务会话 A，再模拟增量回执那样直接把缓存对象交给会话 B。
   - 修复前输出：三处断言全部成立、退出码 1 ——「返回的就是缓存里同一个对象（同引用）」「缓存对象被原地写上了 A 的表」「B 直接读到缓存对象时带着 A 的表（currentSession 按 A 算）」。
   - 修复后输出：三处全部不成立、退出码 0；且 A 仍然在返回的回包里拿到自己的表，挂表行为本身没变。
3. **备选候选的排除**：同批 open issue 里 #1289（界面词表完整性门禁漏检 3 份片段）在当前上游提交上已经被上游自己修好（片段真源已搬到 `config/copy/*.yaml`，门禁改成按目录自动发现，`tests/verify-locale-completeness.ts` 里有明确注释），不是本版可修的缺陷，故未选。

## 根因

`withWorkRouteTable` 的职责是「把一份按当前会话现算的读数挂到回包上」，但它把结果写回了传进来的对象。传进来的对象在缓存短路路径上就是缓存里那份按工作区共享的快照，写回等于把会话相关的字段存进了共享层；正确性从此依赖「每条返回路径都记得先重挂一遍表」这个约定，而增量回执那条路径并不经过重挂，于是约定一破就串会话。

## 改动文件

- `src/host/snapshotEnvelope.ts`（唯一实现改动）：`withWorkRouteTable` 在挂表前先 `Object.assign({}, reply)` 浅拷贝一份，字段写在拷贝上并返回拷贝；读数仍以原回包为输入现算，抛错时仍在拷贝上挂 `{ ok: false, reason: 'read-failed' }`；不是 `ok: true` 的回包、接线处没给读数出口这两种情形仍原样返回、不拷贝。函数上方的说明注释同步补上这条约定与理由。两个调用点（`sessionSnapshot` 的 `attachWorkRouteTable`、`sessionRefresh` 的回包出口）都使用返回值，不依赖原地改写，已逐个核对。
- 版本与出处同步（不改行为）：根 `package.json`（version 1.9.0-beta.10、repository.url 改为本版仓库）、`tests/verify-config-store-package.ts` 里钉住的候选版号、README 与 `docs/README.en.md` 的锁版号、`package/package.json` 与 `package/README.md`（由构建按根清单重新生成）；README 顶部与 CHANGELOG 顶部各加一段本版说明；新增本文件与 `UPSTREAM.md`。

## 复验结果

- 最小复现：修复前退出码 1（三处缺陷断言成立），`git stash` 回到基线复跑仍为 1，恢复修复后为 0。
- 构建：`node scripts/build.mts` 退出码 0。类型检查：改动实现后 `pnpm run typecheck` 退出码 0；版本号改完之后再跑 `pnpm run typecheck` 时，pnpm 会先做一次依赖状态检查并尝试重装，在本沙箱里因 `chown` 不被允许（EPERM）退出码 1——红在装包环节、没走到检查本身；把这条脚本的四个组成步骤（三个生成器 `--check` 加 `scripts/typecheck.mts`）直接用 node 逐个跑，全部退出码 0、结论「全绿」。这是沙箱文件权限的产物，不是代码问题，特此写明以免误读。
- 相关门禁（修复前基线与修复后各跑一遍，结论一致、全绿）：`tests/verify-1082-work-route-table.ts`、`tests/verify-1261-work-route-receipt.ts`（共 82 项）、`tests/verify-1286-snapshot.ts`、`tests/verify-snapshot-cache-first.ts`（8 项）。
- `npm run smoke` 六条冒烟全部通过。实现改动刚落地时跑的 `npm run verify:changed`（当时只选中受影响的门禁）退出码 0；全部改动（含文档与版本钉）就位后再跑一次时，挑选规则按仓库规矩扩大到近全链（97/324 步），其中 1 步红：`tests/verify-587-update-live.ts` 的真宿主判据要求在本机使用范围里找到一份装好且包完好的本插件，本沙箱里没有这样一份安装——已用 `git stash` 回到未改动的基线单独复跑这扇门，同样退出码 1、同一步断言失败，确认是基线既有的环境依赖，与本版改动无关；其余 96 步全部通过。
- 基线既有失败小结：与本版改动直接相关的检查（类型检查、四扇快照/在办表门禁、版本钉门禁、冒烟）在未改动的基线上同样全绿；唯一红的 `verify-587-update-live` 是基线既有的真机环境依赖，已按上面办法对照证明。

## 验证边界

- 没有跑串行全链（`node scripts/verify-chain.ts`）：按仓库自己的规矩（AGENTS.md「跑门禁：先挑档位」），单点改动跑相关门禁加 `verify:changed` 即可；全链留给集成方按需执行。
- 没有在真实 DSH 宿主里做双会话真机复现：缺陷的成立只依赖对象引用与字段写入这两件在代码里可直接读到的事实，离线复现已经把「同一对象、先挂 A 的表、再原样交给 B」这条链完整走通；真机上是否还有别的路径把缓存对象交出去，不影响本版修复对这条链的封堵。
- 本版只封堵「挂表这一步污染共享对象」。`setCache` 按引用存对象、`shortCircuit` 与增量回执按引用返回对象这两件事本身没动：如果将来有别的会话相关字段也往回包上原地写，仍会重蹈覆辙——那属于上游 #1358 里更大范围的加固，不在本版最小改动之内。
