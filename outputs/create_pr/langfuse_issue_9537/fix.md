# PR Title: Add AWS Bedrock Support for LLM-as-a-Judge Evaluations

## Description
This PR implements AWS Bedrock integration for LLM-as-a-Judge evaluations by extending the existing adapter pattern. It adds a new BedrockLLMAdapter, enhances the evaluation service to support Bedrock models, and includes UI components for Bedrock-specific configuration.

## Changes Made

### File: packages/shared/src/adapters/BedrockLLMAdapter.ts
```typescript
// Complete new file
import { BedrockRuntimeClient, InvokeModelCommand } from "@aws-sdk/client-bedrock-runtime";
import { fromEnv, fromIni, fromInstanceMetadata } from "@aws-sdk/credential-providers";
import { LLMAdapter, LLMCallParams, LLMResponse } from "./LLMAdapter";

export interface BedrockConfig {
  provider: 'bedrock';
  region: string;
  modelId: string;
  credentialsType: 'environment' | 'profile' | 'instance';
  profile?: string;
  maxTokens?: number;
  temperature?: number;
  topP?: number;
}

export class BedrockLLMAdapter implements LLMAdapter {
  provider = 'bedrock' as const;
  private client: BedrockRuntimeClient;
  private config: BedrockConfig;

  constructor(config: BedrockConfig) {
    this.config = config;
    this.client = new BedrockRuntimeClient({
      region: config.region,
      credentials: this.getCredentials(config.credentialsType, config.profile),
    });
  }

  private getCredentials(type: string, profile?: string) {
    switch (type) {
      case 'environment':
        return fromEnv();
      case 'profile':
        return fromIni({ profile });
      case 'instance':
        return fromInstanceMetadata();
      default:
        return fromEnv();
    }
  }

  async callModel(params: LLMCallParams): Promise<LLMResponse> {
    const body = this.formatRequestBody(params);
    
    const command = new InvokeModelCommand({
      modelId: this.config.modelId,
      body: JSON.stringify(body),
      contentType: 'application/json',
    });

    try {
      const response = await this.client.send(command);
      const responseBody = JSON.parse(new TextDecoder().decode(response.body));
      
      return this.parseResponse(responseBody);
    } catch (error) {
      throw new Error(`Bedrock API call failed: ${error.message}`);
    }
  }

  private formatRequestBody(params: LLMCallParams) {
    // Handle different Bedrock model formats
    if (this.config.modelId.includes('claude')) {
      return {
        prompt: `\n\nHuman: ${params.prompt}\n\nAssistant:`,
        max_tokens_to_sample: this.config.maxTokens || 1000,
        temperature: this.config.temperature || 0.7,
        top_p: this.config.topP || 1,
      };
    } else if (this.config.modelId.includes('titan')) {
      return {
        inputText: params.prompt,
        textGenerationConfig: {
          maxTokenCount: this.config.maxTokens || 1000,
          temperature: this.config.temperature || 0.7,
          topP: this.config.topP || 1,
        },
      };
    }
    
    throw new Error(`Unsupported Bedrock model: ${this.config.modelId}`);
  }

  private parseResponse(responseBody: any): LLMResponse {
    let content = '';
    
    if (responseBody.completion) {
      // Claude response format
      content = responseBody.completion;
    } else if (responseBody.results?.[0]?.outputText) {
      // Titan response format
      content = responseBody.results[0].outputText;
    } else {
      throw new Error('Unexpected Bedrock response format');
    }

    return {
      content: content.trim(),
      usage: {
        promptTokens: responseBody.usage?.input_tokens || 0,
        completionTokens: responseBody.usage?.output_tokens || 0,
        totalTokens: (responseBody.usage?.input_tokens || 0) + (responseBody.usage?.output_tokens || 0),
      },
    };
  }
}
```

### File: packages/shared/src/adapters/LLMAdapter.ts
```diff
  export interface LLMAdapter {
    provider: string;
+   authenticate?(context?: any): Promise<void>;
    callModel(params: LLMCallParams): Promise<LLMResponse>;
  }

+ export interface LLMResponse {
+   content: string;
+   usage?: {
+     promptTokens: number;
+     completionTokens: number;
+     totalTokens: number;
+   };
+ }

  export interface LLMCallParams {
    prompt: string;
    temperature?: number;
    maxTokens?: number;
+   topP?: number;
  }
```

### File: packages/shared/src/adapters/index.ts
```diff
  export * from './LLMAdapter';
+ export * from './BedrockLLMAdapter';
```

### File: worker/src/features/utils/callLLM.ts
```diff
  import { LLMAdapter } from '@langfuse/shared/src/adapters';
+ import { BedrockLLMAdapter, BedrockConfig } from '@langfuse/shared/src/adapters';

  export async function callLLM(
    params: LLMCallParams,
    modelConfig: any
  ): Promise<LLMResponse> {
    const adapter = createLLMAdapter(modelConfig);
    
+   // Authenticate if needed
+   if (adapter.authenticate) {
+     await adapter.authenticate();
+   }
    
    return await adapter.callModel(params);
  }

  function createLLMAdapter(config: any): LLMAdapter {
    switch (config.provider) {
+     case 'bedrock':
+       return new BedrockLLMAdapter(config as BedrockConfig);
      case 'openai':
        return new OpenAIAdapter(config);
      case 'anthropic':
        return new AnthropicAdapter(config);
      default:
        throw new Error(`Unsupported provider: ${config.provider}`);
    }
  }
```

### File: packages/shared/src/server/llm-api-key/types.ts
```diff
  export type LLMProvider = 
    | 'openai'
    | 'anthropic'
+   | 'bedrock';

+ export interface BedrockCredentials {
+   provider: 'bedrock';
+   region: string;
+   credentialsType: 'environment' | 'profile' | 'instance';
+   profile?: string;
+ }

  export type LLMCredentials = 
    | OpenAICredentials
    | AnthropicCredentials
+   | BedrockCredentials;
```

### File: web/src/components/ModelParameters/index.tsx
```diff
  import { OpenAIParameters } from './OpenAIParameters';
  import { AnthropicParameters } from './AnthropicParameters';
+ import { BedrockParameters } from './BedrockParameters';

  export function ModelParameters({ provider, config, onChange }: ModelParametersProps) {
    switch (provider) {
      case 'openai':
        return <OpenAIParameters config={config} onChange={onChange} />;
      case 'anthropic':
        return <AnthropicParameters config={config} onChange={onChange} />;
+     case 'bedrock':
+       return <BedrockParameters config={config} onChange={onChange} />;
      default:
        return <div>Unsupported provider: {provider}</div>;
    }
  }
```

### File: web/src/components/ModelParameters/BedrockParameters.tsx
```typescript
// Complete new file
import React from 'react';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';

interface BedrockParametersProps {
  config: any;
  onChange: (config: any) => void;
}

const BEDROCK_MODELS = [
  { id: 'anthropic.claude-v2', name: 'Claude v2' },
  { id: 'anthropic.claude-v2:1', name: 'Claude v2.1' },
  { id: 'anthropic.claude-3-sonnet-20240229-v1:0', name: 'Claude 3 Sonnet' },
  { id: 'anthropic.claude-3-haiku-20240307-v1:0', name: 'Claude 3 Haiku' },
  { id: 'amazon.titan-text-express-v1', name: 'Titan Text Express' },
  { id: 'amazon.titan-text-lite-v1', name: 'Titan Text Lite' },
];

const AWS_REGIONS = [
  { id: 'us-east-1', name: 'US East (N. Virginia)' },
  { id: 'us-west-2', name: 'US West (Oregon)' },
  { id: 'eu-west-1', name: 'Europe (Ireland)' },
  { id: 'ap-southeast-1', name: 'Asia Pacific (Singapore)' },
];

export function BedrockParameters({ config, onChange }: BedrockParametersProps) {
  const updateConfig = (key: string, value: any) => {
    onChange({ ...config, [key]: value });
  };

  return (
    <div className="space-y-4">
      <div>
        <Label htmlFor="region">AWS Region</Label>
        <Select value={config.region || ''} onValueChange={(value) => updateConfig('region', value)}>
          <SelectTrigger>
            <SelectValue placeholder="Select AWS region" />
          </SelectTrigger>
          <SelectContent>
            {AWS_REGIONS.map((region) => (
              <SelectItem key={region.id} value={region.id}>
                {region.name}
              </SelectItem>
            ))}
          </SelectContent>
        </Select>
      </div>

      <div>
        <Label htmlFor="modelId">Model</Label>
        <Select value={config.modelId || ''} onValueChange={(value) => updateConfig('modelId', value)}>
          <SelectTrigger>
            <SelectValue placeholder="Select Bedrock model" />
          </SelectTrigger>
          <SelectContent>
            {BEDROCK_MODELS.map((model) => (
              <SelectItem key={model.id} value={model.id}>
                {model.name}
              </SelectItem>
            ))}
          </SelectContent>
        </Select>
      </div>

      <div>
        <Label htmlFor="credentialsType">Credentials Type</Label>
        <Select 
          value={config.credentialsType || 'environment'} 
          onValueChange={(value) => updateConfig('credentialsType', value)}
        >
          <SelectTrigger>
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="environment">Environment Variables</SelectItem>
            <SelectItem value="profile">AWS Profile</SelectItem>
            <SelectItem value="instance">Instance Metadata</SelectItem>
          </SelectContent>
        </Select>
      </div>

      {config.credentialsType === 'profile' && (
        <div>
          <Label htmlFor="profile">AWS Profile</Label>
          <Input
            id="profile"
            value={config.profile || ''}
            onChange={(e) => updateConfig('profile', e.target.value)}
            placeholder="default"
          />
        </div>
      )}

      <div>
        <Label htmlFor="temperature">Temperature</Label>
        <Input
          id="temperature"
          type="number"
          min="0"
          max="1"
          step="0.1"
          value={config.temperature || 0.7}
          onChange={(e) => updateConfig('temperature', parseFloat(e.target.value))}
        />
      </div>

      <div>
        <Label htmlFor="maxTokens">Max Tokens</Label>
        <Input
          id="maxTokens"
          type="number"
          min="1"
          max="4000"
          value={config.maxTokens || 1000}
          onChange={(e) => updateConfig('maxTokens', parseInt(e.target.value))}
        />
      </div>

      <div>
        <Label htmlFor="topP">Top P</Label>
        <Input
          id="topP"
          type="number"
          min="0"
          max="1"
          step="0.1"
          value={config.topP || 1}
          onChange={(e) => updateConfig('topP', parseFloat(e.target.value))}
        />
      </div>
    </div>
  );
}
```

### File: worker/src/features/evaluation/evalService.ts
```diff
  import { callLLM } from '../utils/callLLM';
+ import { BedrockConfig } from '@langfuse/shared/src/adapters';

  export class EvaluationService {
    async executeJudgeEvaluation(job: EvaluationJob): Promise<EvaluationResult> {
+     // Validate provider-specific configuration
+     this.validateProviderConfig(job.modelConfig);
      
      try {
        const response = await callLLM(job.params, job.modelConfig);
        return this.processEvaluationResponse(response, job);
      } catch (error) {
-       throw new Error(`Evaluation failed: ${error.message}`);
+       throw new Error(`Evaluation failed for ${job.modelConfig.provider}: ${error.message}`);
      }
    }

+   private validateProviderConfig(config: any): void {
+     switch (config.provider) {
+       case 'bedrock':
+         this.validateBedrockConfig(config as BedrockConfig);
+         break;
+       // Add other provider validations as needed
+     }
+   }

+   private validateBedrockConfig(config: BedrockConfig): void {
+     if (!config.region) {
+       throw new Error('AWS region is required for Bedrock');
+     }
+     if (!config.modelId) {
+       throw new Error('Model ID is required for Bedrock');
+     }
+     if (!config.credentialsType) {
+       throw new Error('Credentials type is required for Bedrock');
+     }
+   }
  }
```

### File: packages/shared/src/constants.ts
```diff
  export const SUPPORTED_LLM_PROVIDERS = [
    'openai',
    'anthropic',
+   'bedrock',
  ] as const;

+ export const BEDROCK_MODELS = {
+   'anthropic.claude-v2': 'Claude v2',
+   'anthropic.claude-v2:1': 'Claude v2.1',
+   'anthropic.claude-3-sonnet-20240229-v1:0': 'Claude 3 Sonnet',
+   'anthropic.claude-3-haiku-20240307-v1:0': 'Claude 3 Haiku',
+   'amazon.titan-text-express-v1': 'Titan Text Express',
+   'amazon.titan-text-lite-v1': 'Titan Text Lite',
+ } as const;
```

### File: package.json