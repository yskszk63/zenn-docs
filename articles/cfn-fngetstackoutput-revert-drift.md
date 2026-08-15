---
title: "Fn::GetStackOutput の結果が反映されない場合は REVERT_DRIFT で変更セットを作成すると良いかもしれないという話"
emoji: "🙌"
type: "tech"
topics: ["aws", "cloudformation"]
published: false
---

# はじめに (結論)

CloudFormation の `Fn::GetStackOutput` で参照している値が反映されない場合は、`REVERT_DRIFT` (ドリフト認識変更セット) で変更セットを作成すると反映されるかも。

# CloudFormation の CreateChangeSet は二点の比較

たとえば、以下のような CDK App があるとします。

```typescript
#!/usr/bin/env node

import * as cdk from "aws-cdk-lib/core";
import * as ssm from "aws-cdk-lib/aws-ssm";
import * as sfn from "aws-cdk-lib/aws-stepfunctions";

const app = new cdk.App();

const helloMessage = process.env["HELLO_MESSAGE"] ?? "Hello";

const producerStack = new (class extends cdk.Stack {
  readonly param: ssm.IStringParameter;
  constructor() {
    super(app, "ProducerStack", {
      env: {
        region: "ap-northeast-1",
      },
    });

    const param = new ssm.StringParameter(this, "Hello", {
      stringValue: helloMessage,
    });
    this.param = param;
  }
})();

new (class extends cdk.Stack {
  constructor() {
    super(app, "ConsumerStack", {
      env: {
        region: "ap-northeast-1",
      },
    });

    cdk.CrossStackReferences.of(this).consume(cdk.ReferenceStrength.WEAK);

    const states = new sfn.StateMachine(this, "Consumer", {
      definitionBody: sfn.DefinitionBody.fromChainable(
        sfn.Pass.jsonata(this, "Greet", {
          outputs: {
            message: "${message}",
          },
        }),
      ),
      definitionSubstitutions: {
        message: producerStack.param.stringValue,
      },
    });

    new cdk.CfnOutput(this, "StateMachineArn", {
      value: states.stateMachineArn,
    });
  }
})();
```

- `ProducerStack`
    - SSM パラメータを定義
- `ConsumerStack` 
    - クロススタック参照を弱参照に
    - SSM パラメータの値を参照する Step Functions を定義
    - 弱参照のため `Fn::GetStackOutput` で参照

`cdk synth` すると、以下のようになります。

ProducerStack

```yaml
Resources:
  Hello4A628BD4:
    Type: AWS::SSM::Parameter
    Properties:
      Type: String
      Value: Hello
Outputs:
  PublishOutputFnGetAttHello4A628BD4Value6A59DF73:
    Value:
      Fn::GetAtt:
        - Hello4A628BD4
        - Value
...
```

ConsumerStack

```yaml
Resources:
...
  Consumer8D6BE417:
    Type: AWS::StepFunctions::StateMachine
    Properties:
      DefinitionString: '{"StartAt":"Greet","States":{"Greet":{"Type":"Pass","QueryLanguage":"JSONata","Output"
:{"message":"${message}"},"End":true}}}'
      DefinitionSubstitutions:
        message:
          Fn::GetStackOutput:
            StackName: ProducerStack
            Region: ap-northeast-1
            OutputName: PublishOutputFnGetAttHello4A628BD4Value6A59DF73
      RoleArn:
        Fn::GetAtt:
          - ConsumerRoleE2904DDD
          - Arn
    DependsOn:
      - ConsumerRoleE2904DDD
    UpdateReplacePolicy: Delete
    DeletionPolicy: Delete
Outputs:
  StateMachineArn:
    Value:
      Ref: Consumer8D6BE417
...
```

これをデプロイして ConsumerStack の StepFunctions の定義を見てみましょう。

```
$ npx cdk deploy --all
...
$ aws stepfunctions describe-state-machine --state-machine-arn \
  "$(aws cloudformation describe-stacks --stack-name ConsumerStack|jq -r '.Stacks[].Outputs[].OutputValue')" |
  jq -r .definition | jq
{
  "StartAt": "Greet",
  "States": {
    "Greet": {
      "Type": "Pass",
      "QueryLanguage": "JSONata",
      "Output": {
        "message": "Hello"
      },
      "End": true
    }
  }
}
```

次に `ProducerStack` で定義している SSM パラメータの値 `Hello` から `modified` に変更して CDK をデプロイします。

```
$ HELLO_MESSAGE=modified npx cdk deploy --all --yes
...
```

StepFunctions の定義を再度確認してみます。 `.States.Greet.Output.message` の期待値は `modified` です。

```
$ npx cdk deploy --all
...
$ aws stepfunctions describe-state-machine --state-machine-arn \
  "$(aws cloudformation describe-stacks --stack-name ConsumerStack|jq -r '.Stacks[].Outputs[].OutputValue')" |
  jq -r .definition | jq
{
  "StartAt": "Greet",
  "States": {
    "Greet": {
      "Type": "Pass",
      "QueryLanguage": "JSONata",
      "Output": {
        "message": "Hello"
      },
      "End": true
    }
  }
}
```

...変わりませんでした。

AWS のドキュメントには下記のように記載されています。

https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-getstackoutput.html

> NOTE `Fn::GetStackOutput` creates a weak reference. The referenced value is resolved at stack create or update time. If the referenced stack or output is later deleted or modified, the consuming stack is not automatically updated or notified. To ensure consistency, protect referenced stacks and outputs from unintended changes.

デプロイをしないと反映されないためのようですね。では、再度デプロイして確認してみましょう。

```
$ npx cdk deploy --app cdk.out/ ConsumerStack
Including dependency stacks: ProducerStack

✨  Synthesis time: 0.01s

ProducerStack
ProducerStack: creating CloudFormation changeset...
ProducerStack: deploying... [1/2]

✅  ProducerStack (no changes)

✨  Deployment time: 0s

Outputs:
ProducerStack.PublishOutputFnGetAttHello4A628BD4Value6A59DF73 = modified
Stack ARN:
arn:aws:cloudformation:ap-northeast-1:123412341234:stack/ProducerStack/0c3db3d0-9897-11f1-8569-0a33b339da91
ConsumerStack
ConsumerStack: creating CloudFormation changeset...
ConsumerStack: deploying... [2/2]

✅  ConsumerStack (no changes)

✨  Deployment time: 0s

Outputs:
ConsumerStack.StateMachineArn = arn:aws:states:ap-northeast-1:123412341234:stateMachine:Consumer8D6BE417-XTSi0b
FyEiKa
Stack ARN:
arn:aws:cloudformation:ap-northeast-1:123412341234:stack/ConsumerStack/13642590-9897-11f1-8665-0631190941ed

✨  Total time: 2.7s

$ aws stepfunctions describe-state-machine --state-machine-arn \
  "$(aws cloudformation describe-stacks --stack-name ConsumerStack|jq -r '.Stacks[].Outputs[].OutputValue')" |
  jq -r .definition | jq
{
  "StartAt": "Greet",
  "States": {
    "Greet": {
      "Type": "Pass",
      "QueryLanguage": "JSONata",
      "Output": {
        "message": "Hello"
      },
      "End": true
    }
  }
}
```

...変わりませんでした。
ConsumerStack は差分無しとして反映されてません。

CloudFormation のデプロイは下記のようなフローとなっています。

1. CreateChangeSet ... 変更セットを作成
2. ExecuteChangeSet ... 変更セットを実行

`Fn::GetStackOutput` はスタックの作成・更新時に解決されます。
しかし、標準の変更セットではテンプレート間に差分がないため、変更セットを実行するところまで到達できません。
標準の変更セットでは、少なくともこのケースでは、前回デプロイされたテンプレートと今回のテンプレートだけでは差分がないため、変更なしと判定されているように思えます。

同様の問題が SSM Parameter の動的参照 (`{{resolve:ssm:parameter-name:version}}`) でも発生します。
(この問題が嫌なので、最近まで利用を避けてきました。)

# `REVERT_DRIFT` モードは三点で比較

2025 年 11 月ころに CloudFormation に追加されたドリフト対応変更セットは、先の二つに加えて、リソースの現在の状態（Actual state）も比較する3-way comparisonになります。

[アナウンスのブログ - AWS CloudFormation のドリフト対応変更セットで設定のドリフトを安全に処理](https://aws.amazon.com/jp/about-aws/whats-new/2025/11/configuration-drift-enhanced-cloudformation-sets/)

イレギュラーな操作なので、おそらく管理コンソールでの操作が多いのかな (と勝手に想像して) と思うので、ここからは管理コンソールで操作していきます。

対象スタックの「変更セット」タブを開き、「変更セット作成」を押下
![image1](/images/cfn-fngetstackoutput-revert-drift/image1.png)

「変更セットタイプ」を「ドリフト認識変更セット」として「次へ」を押下
![image2](/images/cfn-fngetstackoutput-revert-drift/image2.png)

以降、「次へ」で進み「変更セットをレビュー」画面で「変更セットを作成」を押下
![image3](/images/cfn-fngetstackoutput-revert-drift/image3.png)

しばらく待つと「変更セットのステータス」が「CREATE_COMPLETE」となる。
![image4](/images/cfn-fngetstackoutput-revert-drift/image4.png)

期待通りに差分が検出されているので「変更セット実行」を押下しデプロイされるのを待つ
![image5](/images/cfn-fngetstackoutput-revert-drift/image5.png)

```
$ aws stepfunctions describe-state-machine --state-machine-arn \
  "$(aws cloudformation describe-stacks --stack-name ConsumerStack|jq -r '.Stacks[].Outputs[].OutputValue')" |
  jq -r .definition | jq
{
  "StartAt": "Greet",
  "States": {
    "Greet": {
      "Type": "Pass",
      "QueryLanguage": "JSONata",
      "Output": {
        "message": "modified"
      },
      "End": true
    }
  }
}
```

`Hello` から `modified` に変わりましたね！
なぜこれで `Fn::GetStackOutput` の変更が拾われるのかは AWS のドキュメントからは明確には分かりません。
ただ、 `REVERT_DRIFT` は変更セット作成時にリソースの Actual state を取得して3-way comparisonするため、この過程で参照値が評価されている可能性があります。

# (蛇足) 疑問。CloudFormation の不具合でしょうか...？

一度ふたつのスタックを削除して、問題の状態 (`Hello` -> `modified`: StepFunctions の定義は `Hello` のまま) にして標準変更セットを作成すると、差分は検出されています。
![imageE1](/images/cfn-fngetstackoutput-revert-drift/imageE1.png)

ドリフト検出でも差分検出されています。
![imageE2](/images/cfn-fngetstackoutput-revert-drift/imageE2.png)

この状態で先の変更セットを実行すると、差分が検出されているにもかかわらず反映されません。
![imageE3](/images/cfn-fngetstackoutput-revert-drift/imageE3.png)

もちろん、定義もそのままです。
```
$ aws stepfunctions describe-state-machine --state-machine-arn \
  "$(aws cloudformation describe-stacks --stack-name ConsumerStack|jq -r '.Stacks[].Outputs[].OutputValue')" |
  jq -r .definition | jq
{
  "StartAt": "Greet",
  "States": {
    "Greet": {
      "Type": "Pass",
      "QueryLanguage": "JSONata",
      "Output": {
        "message": "Hello"
      },
      "End": true
    }
  }
}
```

再度変更セットを作成すると、差分無いよ。と言われます。ドリフトは検出できています。
![imageE4](/images/cfn-fngetstackoutput-revert-drift/imageE4.png)

CloudFormation の不具合でしょうか...？変更セット作成時の API 呼び出しのパラメータとテンプレートが一致しているので、テンプレートの比較すらされていないように見えます。
ただ、AWS のサポート契約無いので報告ルートが無いのでここまでで今日はおしまい。
