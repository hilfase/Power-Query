let
    // ============================================================
    // 1. КОНСОЛИДАЦИЯ ИСХОДНЫХ ТАБЛИЦ
    // ============================================================
    Источник = Table.Combine({Source_Install, Source_Pull, Source_Handover, Source_Actual}),

    ПереупорядочитьСтолбцы = Table.ReorderColumns(
        Источник,
        {"Contractor", "Foreman", "Date", "Batch_ID", "Spool_No",
         "DN", "Sheet", "Material", "Material_Code", "Pipe_Length_M"}
    ),

    // ============================================================
    // 2. ФИЛЬТР ПО ПЕРИОДУ И РАСЧЕТ ОБЪЕМА
    // ============================================================
    // Оставляем только текущий и предыдущий месяц
    ФильтрПоПериоду = Table.SelectRows(
        ПереупорядочитьСтолбцы,
        each Date.IsInCurrentMonth([Date]) or Date.IsInPreviousMonth([Date])
    ),

    // Объем = длина трубы × процент выполнения
    ДобавитьОбъем = Table.AddColumn(
        ФильтрПоПериоду,
        "Volume",
        each [Pipe_Length_M] * [Percent_Complete]
    ),

    // Метка типа работы
    ДобавитьТипРаботы = Table.AddColumn(
        ДобавитьОбъем,
        "Work_Type",
        each "Pipeline installation"
    ),

    // ============================================================
    // 3. ОБОГАЩЕНИЕ: СПРАВОЧНИК СТАЛИ
    // ============================================================
    // Левое соединение по коду материала
    ОбъединитьСоСтатью = Table.NestedJoin(
        ДобавитьТипРаботы, {"Material"}, Ref_Steel, {"Material_Key"},
        "steel", JoinKind.LeftOuter
    ),
    РазвернутьСталь = Table.ExpandTableColumn(
        ОбъединитьСоСтатью, "steel", {"Match_Code"}, {"Match_Code"}
    ),

    // ============================================================
    // 4. ФОРМИРОВАНИЕ КЛЮЧА СТЫКА (Joint_ID)
    // ============================================================
    // Если есть номер спула — склеиваем "партия_спул", иначе только партия
    ДобавитьJointID = Table.AddColumn(
        РазвернутьСталь,
        "Joint_ID",
        each if [Spool_No] <> null
            then [Batch_ID] & "_" & [Spool_No]
            else [Batch_ID]
    ),
    // Первые 5 символов ключа — префикс партии для соединения с категорией
    ДобавитьПрефиксПартии = Table.AddColumn(
        ДобавитьJointID, "Batch_Prefix", each Text.Start([Joint_ID], 5)
    ),

    // ============================================================
    // 5. ОБОГАЩЕНИЕ: КАТЕГОРИЯ УЧАСТКА
    // ============================================================
    ОбъединитьСКатегорией = Table.NestedJoin(
        ДобавитьПрефиксПартии, {"Batch_Prefix"}, Ref_Category, {"Batch_Prefix"},
        "category", JoinKind.LeftOuter
    ),
    РазвернутьКатегорию = Table.ExpandTableColumn(
        ОбъединитьСКатегорией, "category", {"Category"}, {"Category"}
    ),

    // ============================================================
    // 6. ОБОГАЩЕНИЕ: ТИТУЛ-ЛИНИЯ
    // ============================================================
    // Соединение по полному ключу стыка
    ОбъединитьСТитулЛинией = Table.NestedJoin(
        РазвернутьКатегорию, {"Joint_ID"}, Ref_TitleLine, {"Joint_ID"},
        "titleline", JoinKind.LeftOuter
    ),
    РазвернутьТитулЛинию = Table.ExpandTableColumn(
        ОбъединитьСТитулЛинией, "titleline",
        {"Elevation_Range", "Wall_Thickness", "Diameter", "Title_Batch"},
        {"Elevation_Range", "Wall_Thickness", "Diameter", "Title_Batch"}
    ),

    // ============================================================
    // 7. ПРИВЕДЕНИЕ ТИПОВ
    // ============================================================
    ПривестиТипТолщины = Table.TransformColumnTypes(
        РазвернутьТитулЛинию, {{"Wall_Thickness", type text}}
    ),
    ПривестиТипы = Table.TransformColumnTypes(
        ПривестиТипТолщины,
        {{"Diameter", type text}, {"Match_Code", type text},
         {"Work_Type", type text}, {"Material_Code", type text},
         {"Material", type text}}
    ),

    // ============================================================
    // 8. ПАРСИНГ ДИАПАЗОНА ВЫСОТНЫХ ОТМЕТОК
    // ============================================================
    // Из строки "от X до Y" достаем минимальную отметку
    ДобавитьОтметкуMin = Table.AddColumn(
        ПривестиТипы, "Elevation_Min",
        each Number.FromText(Text.BetweenDelimiters([Elevation_Range], "from ", " to "))
    ),
    // Достаем максимальную отметку
    ДобавитьОтметкуMax = Table.AddColumn(
        ДобавитьОтметкуMin, "Elevation_Max",
        each Number.FromText(Text.AfterDelimiter([Elevation_Range], " to "))
    ),
    // Пустые отметки заменяем нулем, чтобы избежать ошибок при сравнении
    ЗаменитьNullНаНоль = Table.ReplaceValue(
        ДобавитьОтметкуMax, null, 0, Replacer.ReplaceValue,
        {"Elevation_Min", "Elevation_Max"}
    ),

    // ============================================================
    // 9. ОБОГАЩЕНИЕ: ДИАМЕТР / ТОЛЩИНА
    // ============================================================
    // Составной ключ "код материала - материал"
    ДобавитьКлючDNМатериал = Table.AddColumn(
        ЗаменитьNullНаНоль,
        "DN_Material_Key",
        each [Material_Code] & "-" & [Material]
    ),
    ОбъединитьСДиаметрТолщиной = Table.NestedJoin(
        ДобавитьКлючDNМатериал, {"DN_Material_Key"},
        Ref_DiameterThickness, {"DN_Material_Key"},
        "diathk", JoinKind.LeftOuter
    ),
    РазвернутьДиаметрТолщину = Table.ExpandTableColumn(
        ОбъединитьСДиаметрТолщиной, "diathk",
        {"Diameter", "Batch", "Wall_Thickness"},
        {"Ref_Diameter", "Ref_Batch", "Ref_Thickness"}
    ),

    // ============================================================
    // 10. ОБОГАЩЕНИЕ НОРМАТИВАМИ (два соединения с fallback)
    // ============================================================
    // Первое соединение — по сцепке титул-линии
    ОбъединитьСНормамиПоСцепке = Table.NestedJoin(
        РазвернутьДиаметрТолщину, {"Title_Batch"}, Ref_Norms, {"Batch"},
        "norms", JoinKind.LeftOuter
    ),
    РазвернутьНормыПоСцепке = Table.ExpandTableColumn(
        ОбъединитьСНормамиПоСцепке, "norms",
        {"Rate_Facility_0m", "Rate_Overpass_0m",
         "Rate_Facility_10m", "Rate_Overpass_10m"},
        {"Rate_Facility_0m", "Rate_Overpass_0m",
         "Rate_Facility_10m", "Rate_Overpass_10m"}
    ),
    // Второе соединение — по сцепке из справочника диаметр/толщина
    ОбъединитьСНормамиПоДиаметрТолщине = Table.NestedJoin(
        РазвернутьНормыПоСцепке, {"Ref_Batch"}, Ref_Norms, {"Batch"},
        "norms2", JoinKind.LeftOuter
    ),
    РазвернутьНормыПоДиаметрТолщине = Table.ExpandTableColumn(
        ОбъединитьСНормамиПоДиаметрТолщине, "norms2",
        {"Rate_Facility_0m", "Rate_Overpass_0m",
         "Rate_Facility_10m", "Rate_Overpass_10m"},
        {"Ref_Rate_Facility_0m", "Ref_Rate_Overpass_0m",
         "Ref_Rate_Facility_10m", "Ref_Rate_Overpass_10m"}
    ),

    // ============================================================
    // 11. ВЫБОР ИТОГОВОЙ НОРМЫ
    // Приоритет: данные титул-линии, при отсутствии — справочник Диаметр/Толщина
    // ============================================================
    ДобавитьНормуУстановки0м = Table.AddColumn(
        РазвернутьНормыПоДиаметрТолщине, "Rate_Facility_0m_Eff",
        each if [Rate_Facility_0m] <> null then [Rate_Facility_0m]
             else [Ref_Rate_Facility_0m]
    ),
    ДобавитьНормуЭстакады0м = Table.AddColumn(
        ДобавитьНормуУстановки0м, "Rate_Overpass_0m_Eff",
        each if [Rate_Overpass_0m] <> null then [Rate_Overpass_0m]
             else [Ref_Rate_Overpass_0m]
    ),
    ДобавитьНормуУстановки10м = Table.AddColumn(
        ДобавитьНормуЭстакады0м, "Rate_Facility_10m_Eff",
        each if [Rate_Facility_10m] <> null then [Rate_Facility_10m]
             else [Ref_Rate_Facility_10m]
    ),
    ДобавитьНормуЭстакады10м = Table.AddColumn(
        ДобавитьНормуУстановки10м, "Rate_Overpass_10m_Eff",
        each if [Rate_Overpass_10m] <> null then [Rate_Overpass_10m]
             else [Ref_Rate_Overpass_10m]
    ),

    // ============================================================
    // 12. КЛЮЧ "ДИАМЕТР_ТОЛЩИНА"
    // Приоритет у данных титул-линии, fallback — справочник Диаметр/Толщина
    // ============================================================
    ДобавитьКлючДиаметрТолщина = Table.AddColumn(
        ДобавитьНормуЭстакады10м, "Dia_Thk_Key",
        each if [Rate_Overpass_0m] <> null
            then [Diameter] & "_" & [Wall_Thickness]
            else [Ref_Diameter] & "_" & [Ref_Thickness]
    ),
    ПривестиТипыИтогов = Table.TransformColumnTypes(
        ДобавитьКлючДиаметрТолщина,
        {{"Elevation_Min", type number}, {"Elevation_Max", type number},
         {"Rate_Facility_0m_Eff", type number}, {"Rate_Overpass_0m_Eff", type number},
         {"Rate_Facility_10m_Eff", type number}, {"Rate_Overpass_10m_Eff", type number},
         {"Date", type date}}
    ),

    // ============================================================
    // 13. РАСЧЕТ БАЗОВОЙ НОРМЫ ПО КАТЕГОРИИ И ВЫСОТЕ
    // ============================================================
    // Если обе отметки ≤ 10 м — берем норму для 0 м
    // Если минимальная отметка ≥ 10 м — берем норму для 10 м
    // Категория определяет, использовать норму эстакады или установки
    ДобавитьБазовуюНорму = Table.AddColumn(
        ПривестиТипыИтогов, "Base_Rate",
        each
            if [Elevation_Min] <= 10 and [Elevation_Max] <= 10 then
                if [Category] = "Overpass" then [Rate_Overpass_0m_Eff]
                else if [Category] = "Facility" then [Rate_Facility_0m_Eff]
                else null
            else if [Elevation_Min] >= 10 then
                if [Category] = "Overpass" then [Rate_Overpass_10m_Eff]
                else if [Category] = "Facility" then [Rate_Facility_10m_Eff]
                else null
            else null
    ),

    // ============================================================
    // 14. ИТОГОВЫЙ РАСЧЕТ
    // ============================================================
    // Итог = длина × базовая норма × процент выполнения
    ДобавитьИтогПоНорме = Table.AddColumn(
        ДобавитьБазовуюНорму, "Total_By_Rate",
        each [Pipe_Length_M] * [Base_Rate] * [Percent_Complete]
    ),

    // ============================================================
    // 15. ИСКЛЮЧЕНИЕ ПОДРЯДЧИКОВ И ФИНАЛЬНЫЙ НАБОР СТОЛБЦОВ
    // ============================================================
    ИсключитьПодрядчиков = Table.SelectRows(
        ДобавитьИтогПоНорме,
        each [Contractor] <> "Contractor_A" and [Contractor] <> "Contractor_B"
    ),
    ФинальныеСтолбцы = Table.SelectColumns(
        ИсключитьПодрядчиков,
        {"Contractor", "Foreman", "Date", "Spool_No", "DN", "Sheet",
         "Material", "Material_Code", "Pipe_Length_M", "Stage",
         "Percent_Complete", "Volume", "Category", "Dia_Thk_Key",
         "Base_Rate", "Total_By_Rate"}
    ),

    // ============================================================
    // 16. МАППИНГ ЗОНЫ (РКЦ) ПО ДИАПАЗОНУ DN
    // ============================================================
    // Для каждого DN ищем первую строку в справочнике зон,
    // где DN попадает в диапазон [DN_Min; DN_Max]
    ДобавитьЗону = Table.AddColumn(
        ФинальныеСтолбцы, "Zone_Code",
        each Table.SelectRows(
            Ref_Zone,
            (r) => r[DN_Min] <= [DN] and r[DN_Max] >= [DN]
        ){0}[Zone_Code]?
    )
in
    ДобавитьЗону
