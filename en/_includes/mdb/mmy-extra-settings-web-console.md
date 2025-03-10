- **Backup start (UTC)**

   The time in UTC (in 24-hour format) to start [backing up](../../managed-mysql/operations/cluster-backups.md) the cluster. If the time is not set, the backup will start at 22:00 UTC.

- **Retention period for automatic backups, days**

   Automatic backups are stored for the specified number of days.

- **Maintenance window**: [Maintenance time](../../managed-mysql/concepts/maintenance.md) settings:

   {% include [Maintenance window](console/maintenance-window-description.md) %}

- **Access from {{ datalens-name }}**

   Allows you to analyze cluster data in [{{ datalens-full-name }}](../../datalens/concepts/index.md).

   For more information about setting up a connection, see [Connecting to {{ datalens-name }}](../../managed-mysql/operations/datalens-connect.md).


- **Access from management console**

   It allows you to [execute SQL queries](../../managed-mysql/operations/web-sql-query.md) against the databases in the cluster from the {{ yandex-cloud }} dashboard.


- {% include [datatransfer access](console/datatransfer-access.md) %}

- **Statistics sampling**: Enable this option to use the [{#T}](../../managed-mysql/operations/performance-diagnostics.md) tool in the cluster.

- {% include [Deletion protection](console/deletion-protection.md) %}

   {% include [deletion-protection-limits-db](deletion-protection-limits-db.md) %}
