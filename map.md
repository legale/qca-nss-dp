# NSS Data Plane driver map

## Общее описание
- NSS Data Plane (nss-dp) — это драйвер Ethernet-подсистемы SoC Qualcomm/IPQ (например, IPQ8072), который соединяет MAC/GMAC-контроллеры, PPE-коммутационные блоки и NSS (Network Subsystem) с сетевым стеком Linux.
- Код управляет аппаратным абстрактным уровнем (HAL) для каждого порта, регистрирует `net_device`, настраивает DMA/EDMA, PHY и PPE, а также предоставляет точки расширения для overlay-драйверов, switchdev, ethtool, режимов standby и виртуальных портов (VP).
- В модуле задаются параметры (page_mode, jumbo_mru, budgets и т.д.), глобальный контекст `dp_global_ctx` и массив контекстов плоскости данных `dp_global_data_plane_ctx`, которые используются всеми файлами.

## nss_dp_main.c
**Назначение:** центральный файл драйвера: определяет глобальный контекст, параметры модуля, `net_device_ops`, код инициализации/деинициализации платформы, управление PHY/GMAC, вызовы HAL и dataplane.

**Функции:**
- `nss_dp_do_ioctl(net_device *, ifreq *, int)` — передаёт ioctl в PHY через `phy_mii_ioctl`, если PHY присутствует.
- `nss_dp_change_mtu(net_device *, int)` — программирует новый MTU в GMAC и dataplane; при ошибке откатывает значение.
- `nss_dp_set_mac_address(net_device *, void *)` — валидирует и применяет новый MAC к dataplane и GMAC, используя netdev helper-функции.
- `nss_dp_get_stats64/net_device` вариации — запрашивает статистику из HAL (`getndostats`).
- `nss_dp_xmit(skb *, net_device *)` — передаёт SKB на dataplane (`data_plane_ops->xmit`).
- `nss_dp_close(net_device *)` — последовательное выключение интерфейса: останавливает очередь, уведомляет dataplane о link down, останавливает PHY и GMAC, закрывает dataplane и очищает флажки.
- `nss_dp_open(net_device *)` — инициализирует dataplane (если нужно), включает polling задачки, задаёт offload-фичи, настраивает MAC/MTU, открывает dataplane и GMAC, запускает PHY или сообщает link up.
- `nss_dp_rx_flow_steer(net_device *, sk_buff *, u16 rxq, u32 flow)` — реализует ndo_rx_flow_steer: считывает таблицы RPS, удаляет старое правило и программирует новое через dataplane (`rx_flow_steer`).
- `nss_dp_select_queue()` — выбирает Tx-очередь равную текущему CPU для равномерного распределения; для новых ядер не используется (`__unused`).
- `nss_dp_feature_check(skb *, net_device *, netdev_features_t)` — отключает аппаратный checksum/TCP-segmentation для двукратно VLAN-тегированных кадров на SoC, где это не поддерживается.
- `nss_dp_of_get_pdata(device_node *, net_device *, nss_gmac_hal_platform_data *)` — парсит DT: читает совместимость, MAC ID, PHY, адреса регистров, MAC-адрес, режимы jumbo/page, PPE флаги и т.п., заполняет платформенные данные HAL и приватную структуру драйвера.
- `nss_dp_is_phy_dev(net_device *)` (при switchdev) — проверяет, что netdev обслуживается nss-dp (по ops); используется для фильтрации событий.
- `nss_dp_adjust_link(net_device *)` — callback PHY: синхронизирует состояние канала между PHY, dataplane и сетевым стеком, включая carrier on/off.
- `nss_dp_probe(platform_device *)` — создаёт и регистрирует `net_device`, связывает HAL data_plane ops, настраивает IRQ/ресурсы, регистрирует ethtool/switchdev, инициализирует HAL и dataplane контексты.
- `nss_dp_remove(platform_device *)` — удаляет интерфейс: дерегистрирует netdev, освобождает HAL/ресурсы и очищает глобальный контекст.
- `nss_dp_is_netdev_physical(net_device *)` — ищет netdev в `dp_global_ctx.nss_dp` и сообщает, относится ли он к физическим портам.
- `nss_dp_get_port_num(net_device *)` — возвращает MAC/port ID netdev (или `NSS_DP_INVALID_INTERFACE`).
- `nss_dp_ppeds_get_ops(void)` (при `NSS_DP_PPEDS_SUPPORT`) — пробрасывает PPE-DS operations наружу через HAL helper.
- `nss_dp_nsm_sawf_sc_stats_read(struct nss_dp_hal_nsm_sawf_sc_stats *, u8)` — обёртка вокруг HAL для считывания NSM SAWF статистик по service class.
- `nss_dp_init(void)` — init-модуль: обнуляет глобальный контекст, применяет модульные параметры, вызывает `nss_dp_hal_init()` и регистрирует platform-драйвер.
- `nss_dp_exit(void)` — exit-модуль: дерегистрирует платформенный драйвер и чистит HAL, если init выполнялся.
- `nss_dp_init(void)` (лог) — помимо стандартной инициализации выводит `nss-dp fix-wan-stp build 435f45d marker activated`, затем `nss-dp (fix-wan-stp build 435f45d) module initialized` и `nss-dp: STP bridge guard enabled (marker: fix-wan-stp build 2738045)`, чтобы было невозможно пропустить наш модуль в `dmesg` (`nss_dp_main.c:1174-1271`).
- Кроме функций, файл определяет структуру `nss_dp_netdev_ops`, глобальные параметры (`page_mode`, `jumbo_mru`, budgets, mitigation timers и т.п.) и вспомогательные сущности (mdio data, контексты).

## nss_dp_attach.c
**Назначение:** glue-слой для «overlay» драйверов/модулей, которые хотят перехватить dataplane: содержит API для регистрации, старта/остановки и восстановления дефолтной плоскости данных.

**Функции:**
- `nss_dp_reset_netdev_features(net_device *)` — сбрасывает набор фич netdev, чтобы overlay мог объявить свои.
- `nss_dp_receive(net_device *, sk_buff *, napi_struct *)` — используется overlay-драйверами: проставляет dev/protocol у skb и передаёт пакет в GRO/NAPI.
- `nss_dp_is_in_open_state(net_device *)` — проверяет флаг `__NSS_DP_UP` (netdev поднят ли dataplane).
- `nss_dp_override_data_plane(net_device *, nss_dp_data_plane_ops *, nss_dp_data_plane_ctx *)` — даёт overlay заменить dataplane: закрывает интерфейс, деинициализирует старую плоскость, ставит новые ops/ctx и вызывает их init.
- `nss_dp_start_data_plane(net_device *, nss_dp_data_plane_ctx *)` — после override запускает `ndo_open`, когда overlay готов.
- `nss_dp_restore_data_plane(net_device *)` — откатывает override: закрывает netdev и возвращает стандартные ops/ctx (EDMA).
- `nss_dp_get_netdev_by_nss_if_num(int)` — ищет и возвращает `netdev` по NSS interface number через глобальный контекст.

## nss_dp_eawtp.c
**Назначение:** интеграция с EAWTP (External Automatic Wireless Tuning Platform?) через регистрацию коллбеков, чтобы сообщать число активных портов и реагировать на события линка.

**Функции:**
- `eawtp_get_num_active_ports(void *, eawtp_port_info *)` — проходит по всем nss-dp портам, используя FAL/SSDK для проверки линка (учитывая MHT switch), и заполняет `num_active_port`.
- `eawtp_link_event(notifier_block *, unsigned long, void *)` — notifier от ssdk: при событии считает активные порты и вызывает зарегистрированный callback пользователя.
- `eawtp_port_link_status_ntfy_reg(void *)` / `eawtp_port_link_status_ntfy_unreg(void *)` — регистрируют/дерегистрируют notifier в SSDK.
- `eawtp_nss_unregister_cb(void)` — сбрасывает локально сохранённые указатели на callbacks из регистрационной структуры.
- `eawtp_nss_get_and_register_cb(struct eawtp_reg_info *)` — валидирует и сохраняет `reg_info`, заполняет в ней указатели на локальные функции (подсчёт активных портов и регистрация notifier-ов).

## nss_dp_ethtool_priv.c
**Назначение:** реализация приватных флагов ethtool для управления зеркалированием портов (PPE mirror/analysis).

**Функции:**
- `__nss_dp_get_priv_flags(net_device *)` — отдаёт сохранённые приватные флаги из `nss_dp_dev`.
- `nss_dp_reset_mrr_analysis_port_cfg(uint8_t, uint32_t)` — ищет netdev по номеру порта и сбрасывает указанный флаг (когда анализ-порт переносится).
- `nss_dp_set_analysis_port(net_device *, u32, ppe_drv_dp_mirror_direction_t)` — общая реализация для `эgress/ingress analysis port`: просит PPE driver выставить порт и актуализирует флаги, при необходимости стирая предыдущую конфигурацию.
- `__nss_dp_set_priv_flags(net_device *, u32)` — обработчик запроса ethtool: в зависимости от изменившегося флага настраивает PPE mirroring (ingress/egress порт или analysis-порт), проверяет конфликты и обновляет `ethtool_priv_flags`.

## nss_dp_ethtools.c
**Назначение:** основной набор операций ethtool для интерфейсов nss-dp: статистика, строки, настройки PHY, пауза, EEE, приватные флаги и link settings.

**Функции:**
- `nss_dp_get_ethtool_stats(net_device *, ethtool_stats *, u64 *)` — собирает статистику DMА/EDMA (через dataplane ops) и GMAC MIB (через HAL) и копирует в массив.
- `nss_dp_get_strset_count(net_device *, int)` — количество строк для `get_strings`: приватные флаги или делегирование HAL.
- `nss_dp_get_strings(net_device *, u32, u8 *)` — возвращает строки для ethtool (статистика и приватные флаги).
- `nss_dp_get_settings` / `nss_dp_set_settings` (старые ядра) — проксируют ethtool gset/sset к PHY.
- `nss_dp_get_pauseparam` / `nss_dp_set_pauseparam` — читают и программируют настройку flow-control (pause frames) в `nss_dp_dev` и в PHY.
- `nss_dp_fal_to_ethtool_linkmode_xlate(uint32_t *, uint32_t *)` — helper переводящий флаги FAL EEE в значения ethtool.
- `nss_dp_get_eee` — читает EEE-конфигурацию из FAL (`fal_port_interface_eee_cfg_get`), переводит в ethtool структуру.
- `nss_dp_set_eee` — валидирует запрошенные режимы EEE относительно поддерживаемых, формирует FAL bitmap и программирует его.
- `nss_dp_get_priv_flags` / `nss_dp_set_priv_flags` — thin wrappers вокруг функций из `nss_dp_ethtool_priv.c`.
- `nss_dp_get_ethtool_link_ksetting(net_device *, ethtool_link_ksettings *)` — предоставляет link settings: либо через PHY helper, либо читая фиксированную скорость/дуплекс через FAL для интерфейсов без PHY.
- `nss_dp_set_ethtool_ops(net_device *)` — подвешивает структуру `nss_dp_ethtool_ops`, в которой перечислены все вышеописанные handlers.

## nss_dp_netstandby.c
**Назначение:** интеграция подсистемы DP с «network standby» инфраструктурой: позволяет переводить коммутатор/PPE в пониженное энергопотребление и обратно, а также предоставляет информацию о портах.

**Функции:**
- `nss_dp_netstandby_exit_standby(void *, netstandby_exit_info *)` — вызывает `fal_erp_standby_exit` для основного и (при необходимости) MHT коммутатора, затем сообщает completion callback-у.
- `nss_dp_netstandby_enter_standby(void *, netstandby_entry_info *)` — анализирует список интерфейсов, которые должны остаться активными, формирует битовые карты портов (включая MHT switch) и вызывает `fal_erp_standby_enter`; уведомляет standby subsystem о завершении.
- `nss_dp_get_eth_info(struct nss_dp_eth_netdev_info [], uint8_t)` — заполняет массив сведениями о netdev-ах, статусе MHT-портов и т.д., используя глобальный контекст и SSDK.
- `nss_dp_get_and_register_cb(struct netstandby_reg_info *)` — регистрирует callbacks enter/exit в общем контексте `standby_gbl_ctx` и передаёт указатель app_data обратно в netstandby core.

## nss_dp_switchdev.c
**Назначение:** поддержка switchdev/bridge функциональности, включая настройку STP состояния портов, обработку FDB, фильтрацию slow-protocol, взаимодействие с PPE driver и регистрацию notifiers.

**Функции:**
- `nss_dp_set_slow_proto_filter(nss_dp_dev *, bool)` — программирует PPE ctrlpkt профили, чтобы пропускать STP/LACP slow protocols на отключённых портовых состояниях, отслеживая bitmap активных портов.
- `nss_dp_stp_state_set(nss_dp_dev *, u8)` — переводит STP состояние bridge-порта в эквивалент FAL и вызывает `fal_stp_port_state_set`, при необходимости настраивая slow-proto фильтрацию.
- `nss_dp_attr_get` / `nss_dp_attr_set` — реализации `switchdev_ops` для старых ядер: выдают parent ID, bridge flags и применяют STP state (с защитой VLAN).
- `nss_dp_switchdev_ops`, `nss_dp_switchdev_setup` (старый путь) — вешают switchdev ops на netdev.
- `nss_dp_port_attr_set`, `nss_dp_switchdev_port_attr_set_event` и `nss_dp_switchdev_event` — основной путь для новых ядер: обрабатывают события `SWITCHDEV_PORT_ATTR_SET` (BRIDGE_FLAGS, ageing time, STP state) и оповещают через notifier.
- `nss_dp_bridge_attr_set` (варианты) — при включённом `NSS_DP_SW_BR_OPS` делегирует настройку ageing time и learning в PPE driver; если не поддерживается, возвращает успех без действий.
- `nss_dp_fdb_event` / `nss_dp_switchdev_event_nb` — обслуживают добавление/удаление статических FDB записей через PPE driver (EDMA v2) либо удаление записей (EDMA v1).
- `nss_dp_switchdev_cleanup` / новая версия `nss_dp_switchdev_setup` — регистрируют/дерегистрируют blocking и non-blocking notifier-ы только один раз (`switch_init_done`).
- `nss_dp_netdev_event` — netdevice-notifier, реагирует на `NETDEV_CHANGEUPPER` без `linking`, повторно переводит порт в `BR_STATE_FORWARDING` и пишет `fix-wan-stp: upper removed, forcing forwarding`, чтобы аппарат не оставался в disabled после удаления из моста. Регистрируется через `register_netdevice_notifier`/`unregister_netdevice_notifier` вместе с `switchdev`-notifier-ами (`nss_dp_switchdev.c`).
- `nss_dp_is_bridge_port(net_device *)` — helper возвращает, есть ли у netdev мастера-bridge и пишет `netdev_dbg`, если порт уже не подключён к мосту.
- `nss_dp_attr_set(...)` и `nss_dp_port_attr_set(...)` — перед `nss_dp_stp_state_set()` проверяют `nss_dp_is_bridge_port()`; когда порт уже отвязали от моста, они пишут `netdev_info` (`Skip STP state …`) и игнорируют дальнейшие STP-события, чтобы PPE/FAL не переводил порт в `FAL_STP_DISABLED`.

## nss_dp_vp_main.c
**Назначение:** поддержка виртуальных портов (VP), которые используют dataplane без связанного MAC/GMAC: создаёт фиктивный `net_device`, экспортирует API для регистрации Rx callbacks и отправки пакетов через VP Tx rings.

**Функции:**
- `nss_dp_vp_xmit(net_device *, nss_dp_vp_tx_info *, sk_buff *)` — обёртка, которая даёт dataplane отправить пакет на VP кольца.
- `nss_dp_vp_rx_register_cb(nss_dp_vp_rx_cb_t, nss_dp_vp_list_rx_cb_t)` / `nss_dp_vp_rx_unregister_cb()` — регистрируют/удаляют Rx callbacks (индивидуальные и списковые) с синхронизацией RCU.
- `nss_dp_vp_init(void)` — выделяет и настраивает отдельный `net_device` (macid `NSS_DP_VP_MAC_ID`), цепляет dataplane ops через HAL, вызывает `init/open` dataplane без использования GMAC HAL и добавляет порт в глобальный массив.
- `nss_dp_vp_deinit(net_device *)` — освобождает виртуальный netdev (dataplane остановкой занимается вызывающий код перед тем, как вызвать deinit).
