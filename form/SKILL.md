---
name: form
description: Scaffold and maintain forms using react-hook-form + zod — schemas, field components, payload mappers, validation patterns. Use when creating or modifying forms in features.
allowed-tools: Read, Write, Edit, Glob, Grep
---

# Forms

Form patterns for React 19 + Vite projects using react-hook-form + zod + Shadcn/Radix.

## Architecture

```
features/<Feature>/
├── schemas/
│   └── index.js              # zod schemas + form defaults
├── components/
│   ├── <Feature>Form.jsx     # main form component (RHF context)
│   └── fields/               # small reusable field components
│       ├── NameField.jsx
│       └── StatusField.jsx
└── utils/
    └── index.js              # mappers (toCreatePayload, toUpdatePayload)
```

## Schema Definition

```js
// features/Campaigns/schemas/index.js
import { z } from "zod";

export const campaignFormSchema = z.object({
  name: z.string().min(1, "Name is required").max(200),
  description: z.string().max(1000).optional(),
  status: z.enum(["draft", "active", "paused"]),
  budget: z.coerce.number().min(0, "Budget must be positive").optional(),
  startDate: z.date({ required_error: "Start date is required" }),
  endDate: z.date().optional(),
  tags: z.array(z.string()).default([]),
});

export type CampaignFormValues = z.infer<typeof campaignFormSchema>;

export const campaignFormDefaults = {
  name: "",
  description: "",
  status: "draft",
  budget: undefined,
  startDate: undefined,
  endDate: undefined,
  tags: [],
};
```

## Form Component

```jsx
// features/Campaigns/components/CampaignForm.jsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { campaignFormSchema, campaignFormDefaults } from "../schemas";
import { NameField } from "./fields/NameField";
import { StatusField } from "./fields/StatusField";
import { BudgetField } from "./fields/BudgetField";
import { DateRangeField } from "./fields/DateRangeField";
import { Form } from "~/components/ui/form";

export function CampaignForm({ initialData, onSubmit, onCancel }) {
  const form = useForm({
    resolver: zodResolver(campaignFormSchema),
    defaultValues: initialData ?? campaignFormDefaults,
  });

  const handleSubmit = form.handleSubmit((values) => {
    onSubmit(toCreatePayload(values));
  });

  return (
    <Form {...form}>
      <form onSubmit={handleSubmit} className="space-y-6">
        <NameField control={form.control} />
        <StatusField control={form.control} />
        <BudgetField control={form.control} />
        <DateRangeField control={form.control} />

        <div className="flex justify-end gap-3">
          <Button type="button" variant="outline" onClick={onCancel}>
            Cancel
          </Button>
          <Button type="submit">Save Campaign</Button>
        </div>
      </form>
    </Form>
  );
}
```

## Field Components

```jsx
// features/Campaigns/components/fields/NameField.jsx
import { useFormContext } from "react-hook-form";
import {
  FormField,
  FormItem,
  FormLabel,
  FormControl,
  FormMessage,
} from "~/components/ui/form";
import { Input } from "~/components/ui/input";

export function NameField({ control }) {
  return (
    <FormField
      control={control}
      name="name"
      render={({ field }) => (
        <FormItem>
          <FormLabel>Campaign Name</FormLabel>
          <FormControl>
            <Input placeholder="Summer Sale 2026" {...field} />
          </FormControl>
          <FormMessage />
        </FormItem>
      )}
    />
  );
}
```

```jsx
// features/Campaigns/components/fields/StatusField.jsx
import { useFormContext } from "react-hook-form";
import {
  FormField,
  FormItem,
  FormLabel,
  FormControl,
  FormMessage,
} from "~/components/ui/form";
import { RadioGroup, RadioGroupItem } from "~/components/ui/radio-group";

export function StatusField({ control }) {
  return (
    <FormField
      control={control}
      name="status"
      render={({ field }) => (
        <FormItem>
          <FormLabel>Status</FormLabel>
          <FormControl>
            <RadioGroup
              value={field.value}
              onValueChange={field.onChange}
              className="flex gap-4"
            >
              <RadioGroupItem value="draft" label="Draft" />
              <RadioGroupItem value="active" label="Active" />
              <RadioGroupItem value="paused" label="Paused" />
            </RadioGroup>
          </FormControl>
          <FormMessage />
        </FormItem>
      )}
    />
  );
}
```

## Payload Mappers

```js
// features/Campaigns/utils/index.js
export function toCreatePayload(formValues) {
  return {
    name: formValues.name,
    description: formValues.description || undefined,
    status: formValues.status,
    budget_cents: formValues.budget ? Math.round(formValues.budget * 100) : undefined,
    start_date: formValues.startDate?.toISOString(),
    end_date: formValues.endDate?.toISOString() || null,
    tags: formValues.tags,
  };
}

export function toUpdatePayload(formValues) {
  return {
    ...toCreatePayload(formValues),
    updated_at: new Date().toISOString(),
  };
}
```

## Form with Mutation

```jsx
import { useCreateCampaignMutation } from "../api/mutations/useCampaignMutations";
import { CampaignForm } from "../components/CampaignForm";

function CreateCampaignPage() {
  const mutation = useCreateCampaignMutation();
  const navigate = useNavigate();

  const handleSubmit = async (payload) => {
    await mutation.mutateAsync(payload);
    navigate("/campaigns");
  };

  return (
    <CampaignForm
      onSubmit={handleSubmit}
      onCancel={() => navigate("/campaigns")}
    />
  );
}
```

## Form with Save as Draft (Auto-save)

```jsx
import { useEffect } from "react";
import { useForm } from "react-hook-form";
import { debounce } from "lodash";

function AutoSaveForm({ campaignId }) {
  const form = useForm({ resolver: zodResolver(schema) });

  const save = useCallback(
    debounce(async (values) => {
      await saveDraft(campaignId, values);
    }, 2000),
    [campaignId],
  );

  useEffect(() => {
    const subscription = form.watch((values) => save(values));
    return () => subscription.unsubscribe();
  }, [form.watch]);

  // ... render
}
```

## Best Practices

- Use `z.coerce.number()` for numeric inputs (they return strings)
- `enabled: !!id` on queries that depend on form data
- Keep schemas in `schemas/` — one schema per form, reuse validation across create/edit
- Map form values → API payload in `utils/` — never pass raw form values to mutations
- A field used by two different forms may have two variants — don't force-merge
- Use `form.watch` sparingly — prefer `useWatch` for targeted subscriptions
