# HLF-policy-test
Hyperledger Fabric policy test

This is testcase about hyperledger fabric for policies of each layer(Channel, Application, Orderer.. etc)
Used uploaded `yaml`file `configtx.yaml` for policy test.

- Hyperledger doc
    1) Waht is Policy? 
        Hyperledger Fabric blockchain network is managed **by policies** and policies **reside in the channel configuration**.
        However, policies may also be specified in some other places, such as **chaincodes**.
        If signer or signers of some dats meet some condition required for policies, returned `valid`, otherwise returned `invalid`.

    2) Policy Types
    Policies are encoded in a `common.Policy` message as defined in `fabric/protos/common/policies.proto`. 
        2-1) **SignaturePolicy**
            This policy type is the most powerful. (`AND``OR``NOutOf`)
        2-2) **ImplicitMetaPolicy**
            This policy type is less flexible than SignaturePolicy, and is **only valid in the context of configuration**.
            It aggregates the result of evaluating policies deeper in the configuration hierarchy, which are **ultimately defined by SignaturePolicies**. It **supports good default rules** like “A majority of the organization admin policies”.
    
    3) Configuration and Policies
        To call `Deliver` on the orderer, the signature on the request must satisfy the `/Channel/Readers` policy. However, to gossip a block to a peer will require that the `/Channel/Application/Readers` policy be satisfied.

    4) Constructing a SignaturePolicy
        When evaluating a signature policy against a signature set, signatures are ‘consumed’, in the order in which they appear, regardless of whether they satisfy multiple policy principals.        
        To avoid this pitfall, **identities** should be **specified from most privileged to least privileged** in the policy identities specification, and **signatures** should be **ordered from least privileged to most privileged** in the signature set.

    5) Constructing an ImplicitMetaPolicy
        <u>The `ImplicitMetaPolicy` is only validly defined in the context of channel configuration.</u> `ImplicitMetaPolicy`는 MSP principle을 따르지 않기 때문에 `META`이다.
        rule: `ANY`, `MAJORITY`
        *Note that **policies higher in the hierarchy are all defined as ImplicitMetaPolicys** while leaf nodes necessarily are defined as SignaturePolicys. This set of defaults works nicely because the ImplicitMetaPolicies do not need to be redefined as the number of organizations change, and the individual organizations may pick their own rules and thresholds for what is means to be a Reader, Writer, and Admin.*

    6) Endorsement policies at chaincode-level
        Endorsement policies는 `chaincode-level`과 `key-level` 각각의 hi 설정할 수 있으며, 체인코드 `instantiation`와 `upgrade`를 할 때 `endorsement policy`를 설정할 수 있다.<br>
        ex)<br>
        `peer chaincode instantiate -C <channelid> -n mycc -P "AND('Org1.peer', 'Org2.peer')"`<br>
        **Endorsement policy syntex**<br>
        <ul>`'Org0.admin'`: any administrator of the `Org0` MSP</ul>
        <ul>`'Org1.member'`: any member of the `Org1` MSP</ul>
        <ul>`'Org1.client'`: any client of the `Org1` MSP</ul>
        <ul>`'Org1.peer'`: any peer of the `Org1` MSP</ul>
        **For example:**
        <ul>`AND('Org1.member', 'Org2.member', 'Org3.member')`</ul>
        <ul>`OR('Org1.member', 'Org2.member')`</ul>
        <ul>`OR('Org1.member', AND('Org2.member', 'Org3.member'))`</ul>
        <ul>`OutOf(1, 'Org1.member', 'Org2.member')` is equivalent to `OR('Org1.member', 'Org2.member')`.</ul>
        <ul>`OutOf(2, 'Org1.member', 'Org2.member')` is equivalent to `AND('Org1.member', 'Org2.member')`</ul>
        `key-level`에서의 endorsement policy가 수정되거나 없으면, `chaincode-level`에서 default값으로 setting된다.
        새로운 `key-level` endorsement policies를 설정하기 위해서는 기존의 policy를 따라야 한다. `key-level endorsement policies`가 `chaincode-level endorsement policies`보다 더 우선된다.

    # **ACL(Access Control Lists)**
        ex) Two sample excerpts
        //ACL policy for invoking chaincodes on peer
        `peer/Propose:` `/Channel/Application/Writers`
        //ACL policy for sending block events
        `event/Block:` `/Channel/Application/Readers`
        채널이 이미 구성되어 있는 상태에서 새로운 `policy`를 적용하고 싶다면 `update transactions`를 통해 할 수 있다.

    (**Ref.**: https://hyperledger-fabric.readthedocs.io/en/release-1.4/policies.html)

#`configtx.yaml` Policies 분석
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

//Application layer에서 block에 encoding되는 config transaction이나 genesis.block에 관련된 value에 대한 정의를 할 수 있는 section
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
        //ImplicitMeta는 초기에 configtx.yaml으로 setting하는 것으로 나중에는 변경이 불가하며, 새로운 Policy를 생성할 때도 이 규칙을 따른다(확실치 않음, 더 알아봐야 함)
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