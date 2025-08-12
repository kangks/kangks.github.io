--- 
title: "AWS Malaysia Unicorn Day 2025"
social:
  name: Dr Richard Kang
  links:
    - https://www.linkedin.com/in/kangks/
---

# My Reflections: Accelerating GenAI Adoption - From Idea to Impact

I recently had the privilege of presenting at [Unicorn Malaysia 2025](https://pages.awscloud.com/aws-unicorn-day-my.html), where I shared my perspective on how businesses can turn Generative AI’s potential into production-ready success, supported by real-world examples from our work at [DoiT International](https://doit.com).  

![*Unicorn Malaysia 2025](/assets/images/2025-07-29-AWS-Malaysia-Unicorn-Day-2025-presentation/opening.jpeg)

I began with a sobering statistic from Deloitte’s *[State of Generative AI in the Enterprise Q4 2024](https://www.deloitte.com/us/en/what-we-do/capabilities/applied-artificial-intelligence/content/state-of-generative-ai-in-enterprise.html)*: **“Over two-thirds of GenAI pilots will not be fully scaled in the next three to six months.”** This is a global challenge, not limited to any one geography. Across industries, many organisations are experimenting with GenAI but struggle to move beyond proofs of concept. The barriers are often the same: unclear ROI, challenges in human adoption, and technical or governance hurdles. My focus in the session was to explore how companies can overcome these obstacles and accelerate their journey from idea to impact.

![*Unicorn Malaysia 2025](/assets/images/2025-07-29-AWS-Malaysia-Unicorn-Day-2025-presentation/casestudy1.jpeg)

To bring this to life, I shared four use cases where we helped customers operationalise GenAI:  

- **Overcoming the ROI Fog**: We built a rapid proof of concept using [Amazon Bedrock](https://aws.amazon.com/bedrock/) and the [Nova Sonic](https://aws.amazon.com/ai/generative-ai/nova/speech/) speech-to-speech model to replace human interviewers for candidate screening. This cut costs from around USD 100 per interview to just cents, while enabling instant feedback and 24/7 scalability.  
- **Empowering Teams to Build GenAI**: Through interactive whiteboarding and co-build sessions, we worked with a MarTech customer to design a [Bedrock Agent](https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html) capable of handling over a million clickstreams. We incorporated dense and sparse [Retrieval-Augmented Generation (RAG)](https://aws.amazon.com/what-is/retrieval-augmented-generation/) techniques, creating clarity for go-live and winning internal buy-in.  
- **Unlocking Trust with Clean Data**: For a customer struggling with poor model accuracy, we improved training data quality, fine-tuned using [Guided Reinforcement Policy Optimization (GRPO)](https://builder.aws.com/content/2rJrpj6m2eh591fjMcRZ3ushpB7/deep-dive-into-group-relative-policy-optimization-grpo) with domain-specific data, and distilled the solution into a lightweight model deployed on [Amazon SageMaker AI Inference Endpoint](https://docs.aws.amazon.com/sagemaker/latest/dg/deploy-model.html). This achieved production-grade accuracy, faster inference, and better cost efficiency.  
- **Production-Ready GenAI on AWS**: When a customer faced GPU shortages in their non-AWS cloud, we migrated their GenAI workloads to AWS, implemented [Amazon EKS](https://aws.amazon.com/eks/) with [Karpenter](https://karpenter.sh/) for elasticity, and used [DoiT Flexsave](https://www.doit.com/flexsave-for-compute/) to optimise GPU costs. The result was higher uptime, faster performance, and predictable costs.

The thread across these examples is that success comes from combining speed of execution with strategic readiness—clear ROI targets, robust governance, scalable infrastructure, and disciplined cost management. GenAI can be a powerful growth catalyst, but it requires moving beyond experimentation to operational excellence.

I closed my presentation with an invitation: **“Let’s co-design your next GenAI pilot—anchored in clarity, governance, scalability, and optimised costs. Visit us at our booth or connect via [doit.com](https://doit.com) to turn ideas into impactful execution.”**  

![*Unicorn Malaysia 2025](/assets/images/2025-07-29-AWS-Malaysia-Unicorn-Day-2025-presentation/closing.jpeg)

For me, Unicorn Malaysia 2025 was about sparking action—showing that with the right approach, any business can bridge the gap between GenAI potential and real, lasting impact.
