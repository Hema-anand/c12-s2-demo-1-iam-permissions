| Action | Resource scope | Justification |
|---|---|---|
| `ecr:GetAuthorizationToken` | `*` | Obtain registry authentication; this action does not support an individual repository ARN. |
| `ecr:BatchGetImage` | `application-service` repository ARN | Retrieve image information required for pulling. |
| `ecr:GetDownloadUrlForLayer` | Same repository ARN | Obtain download URLs for image layers. |
| `ecr:BatchCheckLayerAvailability` | Same repository ARN | Check image-layer availability as part of the standard execution-role pull permissions. |
| `logs:CreateLogStream` | Streams in `/ecs/application-service` | Create streams for container logs. |
| `logs:PutLogEvents` | Same log-stream scope | Write application activity and errors. |