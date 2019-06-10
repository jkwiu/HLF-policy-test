# HLF-policy-test
Hyperledger Fabric policy test

This is testcase about hyperledger fabric for policies of each layer(Channel, Application, Orderer.. etc)
Used uploaded `yaml`file `configtx.yaml` for policy test.

- Hyperledger doc
    **1) Waht is Policy?**<br>
        Hyperledger Fabric blockchain network is managed **by policies** and policies **reside in the channel configuration**.<br>
        However, policies may also be specified in some other places, such as **chaincodes**.<br>
        If signer or signers of some dats meet some condition required for policies, returned `valid`, otherwise returned `invalid`.<br>
        **ref**
         <ol>admin cert: chaincode install/instantiate, creating channel 등에 사용</ol><br>
         <ol>signcerts: endorsing을 위한 cert</ol><br>

    **2) Policy Types**<br>
    Policies are encoded in a `common.Policy` message as defined in `fabric/protos/common/policies.proto`. 
        **2-1) SignaturePolicy**<br>
            This policy type is the most powerful. (`AND``OR``NOutOf`)<br>
        **2-2) ImplicitMetaPolicy**<br>
            This policy type is less flexible than SignaturePolicy, and is **only valid in the context of configuration**.<br>
            It aggregates the result of evaluating policies deeper in the configuration hierarchy, which are **ultimately defined by SignaturePolicies**. It **supports good default rules** like “A majority of the organization admin policies”.
    
    **3) Configuration and Policies**<br>
    ```array of policies
    Channel:
    Policies:
        Readers
        Writers
        Admins
    Groups:
        Orderer:
            Policies:
                Readers
                Writers
                Admins
            Groups:
                OrdereringOrganization1:
                    Policies:
                        Readers
                        Writers
                        Admins
        Application:
            Policies:
                Readers
----------->    Writers
                Admins
            Groups:
                ApplicationOrganization1:
                    Policies:
                        Readers
                        Writers
                        Admins
                ApplicationOrganization2:
                    Policies:
                        Readers
                        Writers
                        Admins
        ```
        To call `Deliver` on the orderer, the signature on the request must satisfy the `/Channel/Readers` policy. However, to gossip a block to a peer will require that the `/Channel/Application/Readers` policy be satisfied.<br>
        *Deliver: `channel level`에서 사용됨. 엑세스 제어, 이전에 커밋된 블록에 대한 정보 요청. `commited`된 모든 블록을 원장에게 보냄. `chaincode`에 의해 설정된 `event`가 있으면 블록의 `ChaincodeActionPayload`에서 찾을 수 있다.<br>
    **4) Constructing a SignaturePolicy**<br>
        When evaluating a signature policy against a signature set, signatures are ‘consumed’, in the order in which they appear, regardless of whether they satisfy multiple policy principals.        
        To avoid this pitfall, **identities** should be **specified from most privileged to least privileged** in the policy identities specification, and **signatures** should be **ordered from least privileged to most privileged** in the signature set.<br>
    **5) Constructing an ImplicitMetaPolicy**<br>
        <u>The `ImplicitMetaPolicy` is only validly defined in the context of channel configuration.</u> `ImplicitMetaPolicy`는 MSP principle을 따르지 않기 때문에 `META`이다.
        rule: `ANY`, `MAJORITY`
        *Note that **policies higher in the hierarchy are all defined as ImplicitMetaPolicys** while leaf nodes necessarily are defined as SignaturePolicys. This set of defaults works nicely because the ImplicitMetaPolicies do not need to be redefined as the number of organizations change, and the individual organizations may pick their own rules and thresholds for what is means to be a Reader, Writer, and Admin.*<br>
    **6) Endorsement policies at chaincode-level**<br>
        Endorsement policies는 `chaincode-level`과 `key-level`의 두가지 방법이 있으며, 체인코드의 `instantiation`와 `upgrade`를 할 때 `endorsement policy`를 설정할 수 있다.<br>
        `key-level`에서의 endorsement policy가 수정되거나 없으면, `chaincode-level`에서 default값으로 setting된다.
        새로운 `key-level` endorsement policies를 설정하기 위해서는 기존의 policy를 따라야 한다. `key-level endorsement policies`가 `chaincode-level endorsement policies`보다 더 우선된다.<br>
        ex)<br>
        `peer chaincode instantiate -C <channelid> -n mycc -P "AND('Org1.peer', 'Org2.peer')"`<br>
        **Endorsement policy syntex**<br>
        - `'Org0.admin'`: any administrator of the `Org0` MSP<br>
        - `'Org1.member'`: any member of the `Org1` MSP<br>
        - `'Org1.client'`: any client of the `Org1` MSP<br>
        - `'Org1.peer'`: any peer of the `Org1` MSP<br>
        **For example:**<br>
        - `AND('Org1.member', 'Org2.member', 'Org3.member')`<br>
        - `OR('Org1.member', 'Org2.member')`<br>
        - `OR('Org1.member', AND('Org2.member', 'Org3.member'))`<br>
        - `OutOf(1, 'Org1.member', 'Org2.member')` is equivalent to `OR('Org1.member', 'Org2.member')`.<br>
        - `OutOf(2, 'Org1.member', 'Org2.member')` is equivalent to `AND('Org1.member', 'Org2.member')`<br>
        
# ACL(Access Control Lists)<br>
   **ex)Two sample excerpts**<br>
        ```sample excerpts
        //ACL policy for invoking chaincodes on peer
        peer/Propose:` `/Channel/Application/Writers
        //ACL policy for sending block events
        event/Block:` `/Channel/Application/Readers
        ```
    채널이 이미 구성되어 있는 상태에서 새로운 `policy`를 적용하고 싶다면 `update transactions`를 통해 할 수 있다.

    (**Ref.**: https://hyperledger-fabric.readthedocs.io/en/release-1.4/policies.html)

#`configtx.yaml` Policies excerpts
```Organizations
Organizations:
    -&OrdererOrg
        //block에 접근하기 위해서는 이 Policies의 Readers를 만족하여야 한다
        //canonical path: /Channel/<Application|Orderer>/<OrgName>/<PolicyName>
        Policies:
            Readers:
                //서명방식
                Type: Signature
                Rule: "OR('OrdererMSP.member', 'PeerOrg1MSP.member', 'PeerOrg2MSP.member')"
            Writers:
                Type: Signature
                Rule: "OR('OrdererMSP.member', 'PeerOrg1MSP.member', 'PeerOrg2MSP.member')"
            Admins:
                Type: Signature
                Rule: "OR('OrdererMSP.admin')"
    - &Org1
        //canonical path: /Channel/<Application|Orderer>/<OrgName>/<PolicyName>
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('PeerOrg1MSP.admin', 'PeerOrg1MSP.peer', 'PeerOrg1MSP.client', 'PeerOrg2MSP.peer')"
            Writers:
                Type: Signature
                Rule: "OR('PeerOrg1MSP.admin', 'PeerOrg1MSP.client', 'PeerOrg1MSP.peer')"
            Admins:
                Type: Signature
                Rule: "OR('PeerOrg1MSP.admin')"
    - &Org2
        //canonical path: /Channel/<Application|Orderer>/<OrgName>/<PolicyName>
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('PeerOrg2MSP.admin', 'PeerOrg2MSP.peer', 'PeerOrg2MSP.client', 'PeerOrg1MSP.peer')"
            Writers:
                Type: Signature
                Rule: "OR('PeerOrg2MSP.admin', 'PeerOrg2MSP.client')"
            Admins:
                Type: Signature
                Rule: "OR('PeerOrg2MSP.admin')"

//Application layer에서 block에 encoding되는 config transaction이나 
//genesis.block에 관련된 value에 대한 정의를 할 수 있는 section
Application: &ApplicationDefaults
    Organizations:
        //canonical path: /Channel/Application/<PolicyName>
        Policies:
            Readers:
                Type: ImplicitMeta
                Rule: "ANY Readers"
            Writers:
                Type: ImplicitMeta
                Rule: "ANY Writers"
            Admins:
                Type: ImplicitMeta
                Rule: "MAJORITY Admins"

Orderer: &OrdererDefaults
    Organizations:
    //canonical path: /Channel/Orderer/<PolicyName>
        Policies:
        //ImplicitMeta는 초기에 configtx.yaml으로 setting하는 것으로 나중에는
        //변경이 불가하며, 새로운 Policy를 생성할 때도 이 규칙을 따른다.
        //(확실치 않음, 더 알아봐야 함)
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"
        // BlockValidation specifies what signatures must be included in the block from the orderer for the peer to validate it.
        BlockValidation:
            Type: ImplicitMeta
            Rule: "ANY Writers"

Channel: &ChannelDefaults
    //canonical path: /Channel/<PolicyName>
    Policies:
        // Who may invoke the 'Deliver' API
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        // Who may invoke the 'Broadcast' API
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        // By default, who may modify elements at this config level
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"

Channel: &ChannelTestnet
    //canonical path: /Channel/<PolicyName>
    Policies:
        // Who may invoke the 'Deliver' API
        Readers:
            Type: Signature
            Rule: "OR('PeerOrg1MSP.admin', 'PeerOrg1MSP.peer', 'PeerOrg1MSP.client', 'PeerOrg2MSP.peer')"
        // Who may invoke the 'Broadcast' API
        Writers:
            Type: Signature
            Rule: "OR('OrdererMSP.member', 'PeerOrg1MSP.member', 'PeerOrg1MSP.member')"
        // By default, who may modify elements at this config level
        Admins:
            Type: Signature
            Rule: "OR('PeerOrg1MSP.admin')"
```

1. Testcase
|---|---