
dify 模型管理架构

- 获取供应商列表
api -> service -> manager -> model_factory
```
def get(self):  
    """  
    获取供应商列表  
    """    tenant_id = current_user.current_tenant_id  
    ai_model_provider_service = AIProviderService()  
    provider_list = ai_model_provider_service.get_provider_list(  
        tenant_id=tenant_id  
    )  
  
    return {  
        'data': [marshal(provider, provider_detail_fields) for provider in provider_list],  
        'code': 0,  
        'msg': 'success',  
    }
```

```
def get_provider_list(self, tenant_id: str, model_type: Optional[str] = None) -> list[ProviderResponse]:  
    """  
    get provider list.  
    :param tenant_id: workspace id    :param model_type: model type    :return:    """    try:  
        # 获取所有供应商  
        provider_configurations = self.provider_manager.get_configurations(tenant_id)  
  
        provider_responses = []  
        for provider_configuration in provider_configurations.values():  
            if model_type:  
                model_type_entity = ModelType.value_of(model_type)  
                if model_type_entity not in provider_configuration.provider.supported_model_types:  
                    continue  
  
            custom = provider_configuration.provider.provider_type == ProviderType.CUSTOM.value  
  
            if custom:  
                proxy_url = provider_configuration.custom_configuration.provider.credentials.get('proxy_url')  
                icon = provider_configuration.provider.icon_small  
            else:  
                proxy_url = provider_configuration.get_custom_credentials(obfuscated=True)  
                icon = provider_configuration.provider.icon_small.en_US  
            provider_responses.append(  
                {  
                    'provider_id': provider_configuration.provider.provider_id,  
                    'provider_name': provider_configuration.provider.provider,  
                    'description': provider_configuration.provider.description,  
                    'provider_type': provider_configuration.provider.sdk_type,  
                    'proxy_url': proxy_url,  
                    "icon": icon,  
                    "enabled": provider_configuration.system_configuration.enabled,  
                    "custom": custom  
                }  
            )  
        return provider_responses
```

```
class ProviderManager:  
    """  
    ProviderManager is a class that manages the model providers includes Hosting and Customize Model Providers.    """  
    def __init__(self) -> None:  
        self.decoding_rsa_key = None  
        self.decoding_cipher_rsa = None  
  
    def get_configurations(self, tenant_id: str) -> ProviderConfigurations:  
        provider_name_to_provider_records_dict = self._get_all_providers(tenant_id)  
  
        # Initialize trial provider records if not exist  
        provider_name_to_provider_records_dict = self._init_trial_provider_records(  
            tenant_id,  
            provider_name_to_provider_records_dict  
        )  
  
        # Get all provider model records of the workspace  
        # 获取model表所有配置信息  
        provider_name_to_provider_model_records_dict = self._get_all_provider_models(tenant_id)  
  
        # 获取内置的所有供应商和模型  
        # Get all provider entities  
        provider_entities = model_provider_factory.get_providers()  
  
        # Get All preferred provider types of the workspace  
        provider_name_to_preferred_model_provider_records_dict = self._get_all_preferred_model_providers(tenant_id)  
  
        # 读取model-settings 表  
        # Get All provider model settings  
        provider_name_to_provider_model_settings_dict = self._get_all_provider_model_settings(tenant_id)  
  
        # Get All load balancing configs  
        provider_name_to_provider_load_balancing_model_configs_dict \  
            = self._get_all_provider_load_balancing_configs(tenant_id)  
  
        # 创建一个空的供应商配置对象  
        provider_configurations = ProviderConfigurations(  
            tenant_id=tenant_id  
        )  
  
        # 获取内置供应商的信息  
        # Construct ProviderConfiguration objects for each provider  
        for provider_entity in provider_entities:  
  
            # handle include, exclude  
            if is_filtered(  
                    include_set=config.POSITION_PROVIDER_INCLUDES_SET,  
                    exclude_set=config.POSITION_PROVIDER_EXCLUDES_SET,  
                    data=provider_entity,  
                    name_func=lambda x: x.provider,  
            ):  
                continue  
  
            provider_name = provider_entity.provider  
            provider_records = provider_name_to_provider_records_dict.get(provider_entity.provider, [])  
            provider_model_records = provider_name_to_provider_model_records_dict.get(provider_entity.provider, [])  
  
            # Convert to custom configuration  
            custom_configuration = self._to_custom_configuration(  
                tenant_id,  
                provider_entity,  
                provider_records,  
                provider_model_records  
            )  
  
            # Convert to system configuration  
            system_configuration = self._to_system_configuration(  
                tenant_id,  
                provider_entity,  
                provider_records  
            )  
  
            # Get preferred provider type  
            preferred_provider_type_record = provider_name_to_preferred_model_provider_records_dict.get(provider_name)  
  
            if preferred_provider_type_record:  
                preferred_provider_type = ProviderType.value_of(preferred_provider_type_record.preferred_provider_type)  
            elif custom_configuration.provider or custom_configuration.models:  
                preferred_provider_type = ProviderType.CUSTOM  
            elif system_configuration.enabled:  
                preferred_provider_type = ProviderType.SYSTEM  
            else:  
                preferred_provider_type = ProviderType.CUSTOM  
  
            using_provider_type = preferred_provider_type  
            has_valid_quota = any(quota_conf.is_valid for quota_conf in system_configuration.quota_configurations)  
  
            if preferred_provider_type == ProviderType.SYSTEM:  
                if not system_configuration.enabled or not has_valid_quota:  
                    using_provider_type = ProviderType.CUSTOM  
  
            else:  
                if not custom_configuration.provider and not custom_configuration.models:  
                    if system_configuration.enabled and has_valid_quota:  
                        using_provider_type = ProviderType.SYSTEM  
  
            # Get provider load balancing configs  
            provider_model_settings = provider_name_to_provider_model_settings_dict.get(provider_name)  
  
            # Get provider load balancing configs  
            provider_load_balancing_configs \  
                = provider_name_to_provider_load_balancing_model_configs_dict.get(provider_name)  
  
            # Convert to model settings  
            model_settings = self._to_model_settings(  
                provider_entity=provider_entity,  
                provider_model_settings=provider_model_settings,  
                load_balancing_model_configs=provider_load_balancing_configs  
            )  
  
            provider_configuration = ProviderConfiguration(  
                tenant_id=tenant_id,  
                provider=provider_entity,  
                preferred_provider_type=preferred_provider_type,  
                using_provider_type=using_provider_type,  
                system_configuration=system_configuration,  
                custom_configuration=custom_configuration,  
                model_settings=model_settings  
            )  
  
            provider_configurations[provider_name] = provider_configuration
```



- 获取自定义供应商的自定义模型列表
``
```
"""  
获取模型列表  
"""  
tenant_id = current_user.current_tenant_id  
  
model_provider_service = AIProviderService()  
models = model_provider_service.get_models_by_provider(  
    tenant_id=tenant_id,  
    provider_id=provider_id,  
    _model_type=None  
)
```


```

```