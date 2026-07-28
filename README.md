```java

@Component
class InitiativeMetricApplicabilityPolicy {

    fun isApplicable(
        metric: MetricsDirectoryEntity,
        agentType: InitiativeMetricAgentType,
    ): Boolean =
        when (agentType) {
            InitiativeMetricAgentType.AUTONOMOUS ->
                metric.autonomousApplicability == true

            InitiativeMetricAgentType.COPILOT ->
                metric.copilotApplicability == true

            InitiativeMetricAgentType.APPEALS ->
                metric.requiresAppealsWork == true
        }

    fun findApplicableAgentTypes(
        metric: MetricsDirectoryEntity,
        requestedAgentTypes: Set<InitiativeMetricAgentType>,
    ): Set<InitiativeMetricAgentType> =
        requestedAgentTypes
            .filterTo(mutableSetOf()) { agentType ->
                isApplicable(
                    metric = metric,
                    agentType = agentType,
                )
            }
}



```
