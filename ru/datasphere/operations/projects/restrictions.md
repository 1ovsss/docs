# Задать ограничения для проекта

{% include [serverless-deprecation-note](../../../_includes/datasphere/serverless-deprecation-note.md) %}

В {{ ml-platform-name }} вы можете настроить ограничения потребления для проекта. Пороги потребления для проекта задаются в [юнитах](../../pricing.md#unit).

1. {% include [include](../../../_includes/datasphere/ui-find-project.md) %}
1. Перейдите на вкладку **{{ ui-key.yc-ui-datasphere.project-page.tab.settings }}** и в блоке **{{ ui-key.yc-ui-datasphere.project-page.settings.limits }}** нажмите кнопку ![pencil](../../../_assets/console-icons/pencil-to-line.svg) **{{ ui-key.yc-ui-datasphere.common.edit }}**.
1. Задайте одно или несколько ограничений для вычислений в проекте:

   * **{{ ui-key.yc-ui-datasphere.edit-project-page.balance }}** — общее количество юнитов, доступных для проекта. Каждое исполнение ячейки будет уменьшать баланс на то количество юнитов, которое необходимо для выполнения одной секунды вычислений выбранной [конфигурации](../../concepts/configurations.md). Ячейки можно запускать до тех пор, пока баланс положительный. Если во время вычислений в одной из ячеек баланс станет меньше или равен нулю, все запущенные вычисления будут остановлены с предупреждением, что баланса проекта недостаточно.
   
   * **{{ ui-key.yc-ui-datasphere.edit-project-page.per-cell }}** — сколько юнитов может потратить одна ячейка в проекте. Если в процессе вычислений ячейка превысит установленное ограничение, она будет остановлена.

   * **{{ ui-key.yc-ui-datasphere.edit-project-page.per-time }}** — сколько юнитов в час может потратить проект. При запуске вычислений проводится проверка, и если [цена за час](../../pricing.md#prices) для выбранной конфигурации превышает заданное значение, ячейка не запустится. Если в процессе вычислений в ячейке текущее потребление проекта за час превысит это значение, все ячейки с вычислениями будут остановлены.

1. Нажмите кнопку **{{ ui-key.yc-ui-datasphere.common.save }}**.

В блоке [{{ ui-key.yc-ui-datasphere.common.general }}](update.md) вы можете указать параметр **{{ ui-key.yc-ui-datasphere.edit-project-page.period-of-inactivity }}** — как скоро перестанут выполняться ячейки с загрузкой CPU или GPU меньше 1% (по умолчанию — `{{ ui-key.yc-ui-datasphere.common.never }}`).

#### См. также {#see-also}

* [{#T}](install-dependencies.md).
* [{#T}](control-compute-resources.md).
